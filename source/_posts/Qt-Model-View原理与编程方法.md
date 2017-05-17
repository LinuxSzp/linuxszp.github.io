---
title: Qt-Model/View原理与编程方法
date: 2016-09-01 20:48:37
categories: [Qt]
tags: [Qt, MVF]
---

由于经常需要将数据库中的数据已不同的形式显示出来，所以使用Qt的Model/View框架会比较多，特意去读了一下源码，总结一下。 

本文将先对Model/View框架中的组件，以及组件之间的关系做一个简单的介绍。
其次，还会介绍以下内容：
 - Model/View框架类中的成员函数是怎么相互协调工作的
 - 通过源码分析，理解执行过程

如果你之前编译动态库时，从未考虑过**二进制兼容**问题，那你就有必要学习一下Qt源码中是怎么解决这个问题的了。

如果你还未曾读过Qt的源码，如果你想尝试去学习一些Qt源码中的一些编程方法，希望这篇博客的源码分析部分能给你带来一点读懂源码的灵感。

<!-- more -->

## **Model/View框架介绍** ##

  Model/View框架实现了将数据和界面分离，使同一数据源可以在界面上以不同的表现形式显示出来(表格、树状或列表)。
  主要有3部分组成，模型(Model)、视图(View)和委托(Delegate)。它们之间的关系为:

 ![qmv1](/img/qmv1.png) 

 - *只有*模型(Model)可以*直接*访问数据
 - 视图(View)和委托(Delegate)通过模型(Model)来获取数据
 - 视图(View)可以通过模型(Model)对数据渲染，也可以通过委托(Delegate)进行编辑和渲染

### **模型(Model)** ### 

  模型为视图和委托提供了访问数据的接口，模型通过一个root节点，为数据建立层次结构。

  每一个子节点都指向各自的父节点，也可以作为下一个节点的父节点，类似与树结构，同时为每一个子节点指定一种模型，并为结构中的数据建立索引。这里的子节点也就是数据项。视图和委托根据索引来间接访问数据。当视图设置一种模型后进行显示时，通过索引和父节点找到该模型下的数据进行显示。

  模型中的数据项并不单单指的是数据的值，一个数据项有角色(role)和值组成。角色使数据项变的多元化，模型通过角色来告诉视图应该如何显示这个数据项，比如字体大小应为多大、字体的颜色、背景等。视图和委托可以通过向模型指定数据项的的索引和角色，获得角色对应的值。

 常用的角色有：

| 常量              | 对应的角色                            | 值类型                |
| ------------------|:--------------------------------------| ---------------------:|
| Qt::DisplayRole   | 代表一个数据项的文字                  | QString或数值类型     |
| Qt::DecorationRole| 代表一个数据项的图标                  | QColor,QIcon或QPixmap |             
| Qt::EditRole      | 一个数据项进行编辑时所要处理的数据    | QString或其他         |
| Qt::FontRole      | 数据项对应的字体                      | QFont                 |
| Qt::BackgroundRole| 数据项背景                            | QBrush                |
| Qt::ForegroundRole| 字体颜色                              | QBrush                |
| Qt::CheckStateRole| 一个数据项的复选状态                  | Qt::CheckState        |


## **源码分析** ##
 
```bash
QStandardItemModel *model = new QStandardItemModel(this);
QTableView *view = new QTableView(this);

view->setModel(model);

model->setHorizontalHeaderItem(0, new QStandardItem(QString::fromUtf8("功能号")));
model->setHorizontalHeaderItem(0, new QStandardItem(QString::fromUtf8("名称")));

model->setItem(0, 0, new QStandardItem(QString::number(1)));
model->setItem(0, 1, new QStandardItem(QString::fromUtf8("源码分析")); 

view->show();
```
  
  相信如果使用Model/View框架时，与上面相似的代码会被使用。接下来将分析源码，在这些代码的背后，模型类和视图类都做了什么工作，模型如何创建了数据了层次结构，视图如何通过模型将数据显示出来的。
  
  查看Qt源码可以访问这个网站：[https://code.woboq.org](https://code.woboq.org)

### **Qt源码中编写习惯** ###

 在这里简单介绍一下Qt中的d-pointer技术。
 
 详细的介绍可以看[https://wiki.qt.io/D-Pointer](https://wiki.qt.io/D-Pointer)

 当一个公共类需要一些成员变量时，我们一般的做法是：
 ```bash
 Class CardOP {
    ....
    
    private:
        QString sName;
        QString sID;
 }
 ```
 直接将成员变量写在公共类中，而Qt源码中很少会这样写，除非已经确定一个类以后不会再添加额外的成员变量。
 
  在编译动态库时，如果我们直接将成员变量写在了类中，当应用程序使用这个库编译发布程序部署后，如果对这个库进行了升级(类中添加了新的成员变量、虚函数或新的功能等)，如果直接去替换老的动态库而不对程序进行重新编译，那么轻则运行不正常，重则直接崩溃。

  为什么会出现这种问题，这就是**二进制兼容性**问题。理想状态下，当我们部署过一个应用程序后，如果后期动态库升级从一个版本升级到另一个版本时，不需要重新编译应用程序就可以直接运行。

  之所以在升级动态库可能会打破**二进制兼容**，c＋＋编译器生成代码的时候，会使用*偏移量*来访问对象的数据，当升级动态库增加新的成员变量的时候，尤其是新增虚函数，使内部布局发生改变，直观的表现是使用sizeof的时候类的大小发生了改变。
  虚函数是通过虚函数表来调用的，增加一个虚函数，使以前存在的虚函数在虚函数表‘位置’发生了改变，升级后，应用程序调用同样的函数，会认为还在原来的位置，而实际上位置已经发生了变化，读了不该读的内存，使程序运行不正常或崩溃。
  
  Qt使用d-pointer技术很好的解决了**二进制兼容**问题。在Qt源码中会看到Q_D，Q_Q宏，以及后缀为Private的类，Q_D，Q_Q就是所说的d-pointer。其主要的目的就是保持公共类的大小保持不变，将一个公共类的成员变量和方法放在自己的Private类中，在公共类中定义一个d_ptr指针指向Private类，同时在Private类中定义一个q_ptr指针用来指向自己的公共类，这样双方都可以互访。将Private类定义成不可见(在.cpp文件中定义和实现，在.h文件中声明)，只会在公共类通过set和get函数访问自己的成员变量，即可以隐藏类公共类的实现细节，又保证了二进制兼容。
  举个例子：
 ```bash
 //CardOP.cpp
 Class CardOPPrivate {
 public:
 	CardOPPrivate(CardOP *q) : q_ptr(q) {
 	}
 	//方法
 private:
 	CardOP *q_ptr;
 
 	QString sName;
 	QString sID;
 }
 
 //CardOP.h
 class CardOPPrivate;
 Class CardOP{
 	CardOP() : d_ptr(new CardOPPrivate(this)) {
 	}
 	
 private:
 	CardOPPrivate *d_ptr;
 }
 ```
 这样编译后公共类CardOP的布局(成员位置也要不变)或大小是不会发生改变的，因为指针的大小是固定的，使用前置声明，确保了即使修改了Private也可以保证公共类的内存空间布局不会发生改变，达到二进制兼容。
 
### **模型代码分析** ###

 分析模型前，先看一看下图中类之间的关系：
 
  ![qmv2](/img/qmv2.png)
  
 公共类的成员变量和方式使用了上面介绍的d-pointer的方法。类图中的d_ptr和q_ptr指针的作用和上面介绍的作用相同，从类图中可以看出QObject中有一个保护成员d_ptr指针，QStandardItemModel间接继承类QObject，所以QStandardItemModel是可以访问的。同样的，QStandardItemModelPrivate也可以访问QOjectData中的q_ptr指针。
 
#### 创建一个模型 ####
当使用
```bash
QStandardItemModel *model = new QStandardItemModel(this);
``` 
创建一个模型时，通过
```bash
QStandardItemModel::QStandardItemModel(QObject *parent)
	    : QAbstractItemModel(*new QStandardItemModelPrivate, parent)
{
	Q_D(QStandardItemModel);
	d->init();
	d->root->d_func()->setModel(this);
}
```
从类图中可以看到QAbstractItemModel和QObject都有一个保护的构造函数，它们的作用就是使QObject中的d_ptr指向创建的QStandardItemModelPrivate，最后到QObject中的代码为
```bash
QObject::QObject(QObjectPrivate &dd, QObject *parent)
	    : d_ptr(&dd)
{
	Q_D(QObject);
	d_ptr->q_ptr = this;
	
	....
	
}
```
可以看到将d_ptr的地址指向了dd，也就是QStandardItemModel构造函数中通过new分配的QStandardItemModelPrivate，当QStandardItemModel需要使用QStandardItemModelPrivate则使用Q_DECLARE_PRIVATE(QStandardItemModel)先得到d_ptr指针，再使用Q_D将d_ptr强转成QStandardItemModelPrivate类型。
在QObject构造函数中有一个`d_ptr->q_ptr = this`使q_ptr地址指向了自身，当QStandardItemModelPrivate需要访问QStandardItemModel时，可以使用Q_DECLARE_PUBLIC(QStandardItemModel)先得到q_ptr，再使用Q_Q将其强转成QStandardItemModel类型。

QStandardItemModelPrivate中有一个QStandardItem类型的root节点，它就是在介绍模型的时候所说的root根节点，QStandardItem同样有d_ptr和q_ptr指针，root中的children成员用于保存以它为父节点的子节点(数据项)，value用来保存数据项中不同角色的值，parent用来指定父节点，model用来指定所属的模型。每一个子节点都与root拥有相同的设置，因为都是同一种类型QStandardItem。当然root节点并不需要有角色和角色值，因为它本身对应用程序来说是不可见的。

接下来介绍QStandardItemModelPrivate中root中的值，通过new一个QStandardItemModel的同时，也创建了一个QStandardItemModelPrivate，通过QStandardItemModelPrivate的构造函数
```bash
QStandardItemModelPrivate::QStandardItemModelPrivate()
	    : root(new QStandardItem),
	      itemPrototype(0),
	      sortRole(Qt::DisplayRole)
{
	root->setFlags(Qt::ItemIsDropEnabled);
}
```
为root创建一个QStandardItem，调用QStandardItem的构造函数
```bash
QStandardItem::QStandardItem()
	    : QStandardItem(*new QStandardItemPrivate)
{
}
```
跳转到了QStandardItem(*new QStandardItemPrivate)保护构造函数中
```bash
QStandardItem::QStandardItem(QStandardItemPrivate &dd)
	    : d_ptr(&dd)
{
	Q_D(QStandardItem);
	d->q_ptr = this;
}
```
为root创建了一个QStandardItemPrivate，并将root中的d_ptr指向分配的QStandardItemPrivate，同时将分配的QStandardItemPrivate中的q_ptr指向root。通过调用QStandardItemPrivate的构造函数对root中的成员进行初始化。
```bash
inline QStandardItemPrivate()
	        : model(0),
	          parent(0),
	          rows(0),
	          columns(0),
	          q_ptr(0),
             lastIndexOf(2)
	        { }
```
通过构造函数一切初始完之后，QStandardItemModel调用d->init()绑定dataChanged信号，进而发送itemChanged信号。d->root->d_func()->setModel(this)设置了root所属的模型，为自身的这种模型。

#### 模型为数据项创建层次结构 ####

模型通过setItem添加数据时，实际上就是为root添加子节点
```bash
void QStandardItemModel::setItem(int row, int column, QStandardItem *item)
{
	Q_D(QStandardItemModel);
	d->root->d_func()->setChild(row, column, item, true);
}
```
可以看看在setChild函数中，都做了什么动作
```bash
void QStandardItemPrivate::setChild(int row, int column, QStandardItem *item,
	                                    bool emitChanged) 
{
	Q_Q(QStandardItem);		//得到root根节点
	
	....
	
	if (rows <= row)
		q->setRowCount(row + 1);	//改变行总数
	if (columns <= column)
		q->setColumnCount(column + 1);	//改变列总数
	int index = childIndex(row, column);	//得到对应行和列的一个下标值
	
	....
	
	if (item) {
		//新增加的节点这个条件为真，在介绍创建一个root时已经讲过分配一个QStandardItem时其中成员的初始值
		if (item->d_func()->parent == 0) {  
			//将这个节点的父节点指向root，同时设置该节点的model与root的model一样
			item->d_func()->setParentAndModel(q, model);
		} else {
			....
		}
	}
	
	....
	
	children.replace(index, item);		//将节点添加到root孩子节点中的相应下标中
	
	....
	
	if (model && emitChanged)
		emit model->layoutChanged();	//发送一个布局改变信号
	
	if (emitChanged && model)
		model->d_func()->itemChanged(item);	//在这里面会发送一个dataChanged信号
}
```
这样就通过root节点将各个节点组成了可以认为是树的一种结构，只是存放在了一个QVertor中，在查找节点时，先通过节点的parent找到父节点，然后通过行和列(或索引)转换成root中children中的下标，从而得到节点。
model发送layoutChanged信号，如果视图使用setModel()设置过模型后，视图会接收到模型发过来的layoutChanged信号，去执行视图的一个内部槽q_layoutChanged()。

### **视图代码分析** ###

 在分析视图之前，同样先看一下类图，如图
 
 ![qmv3](/img/qmv3.png)
 
 可以看出同样的要绑定d_ptr和q_ptr指针，绑定的方法在前面分析模型的时候已经介绍过，这里就不再介绍了。

#### 创建一个视图 ####
```bash
QTableView *view = new QTableView(this);

view->setModel(view);

view->show();
```
通过new分配一个QTableView时，同样是通过父类的保护构造函数将分配的QTableViewPrivate层层向上传递，直到传到QObject，对d_ptr和q_ptr进行指定，在这里关键处是传递到QAbstractItemView时，
```bash
QAbstractItemView::QAbstractItemView(QAbstractItemViewPrivate &dd, QWidget *parent)
    : QAbstractScrollArea(dd, parent)
{
	d_func()->init();
}
```
有一个初始化函数init()，实现在QAbstractItemViewPrivate中，从类图中也可以看出来，在init()中
```bash
QAbstractItemViewPrivate::init()
{
	Q_Q(QAbstractItemView);
	q->setItemDelegate(new QStyledItemDelegate(q));
	
	....
	
}
```
在setItemDelegate中，为视图创建了一个默认的委托QStyledItemDelegate，并将其赋值给了itemDelegate，同时绑定了commitData，closeEditor信号到对应的槽。如果视图可以编辑，则编辑完后，会通过commitData信号到对应的槽中将修改模型中的数据，这过程会在后面介绍，视图是怎么一步步将数据传给模型的。在这里需要有个印象默认创建的委托是QStyledItemDelegate类型。

#### 为视图设置模型 ####

 当使用 `view->setModel(model)` 为视图设置一个模型的时候，其主要工作就是将QTableViewPrivate中的model变量赋值成传入的model，这样可以视图就可以通过QTableViewPrivate中的model找到同为这个model的数据，同时绑定一些信号和槽，因为基本都是使用connect进行绑定信号的，所以代码就不贴出来了，可以自己去看源码。需要注意的是，QTableViewPrivate中的model的初始值为QAbstractItemModelPrivate::staticEmptyModel()，这是在new一个QTableView的时候做的。
 
#### 显示 ####

视图调用show()进行显示时，会通过paintEvent绘制出来的，如果你之前有过写打印功能的代码，实现原理与打印时预览界面绘制的方法基本相同，知道应为几行几列，绘制表格，将数据填到表格中，如果想细读的话，可以按照这条主线去理解，这里我要说的是在paintEvent函数中的
```bash
	const QModelIndex index = d->model->index(row, col, d->root);
	if (index.isValid()) {
		option.rect = QRect(colp + (showGrid && rightToLeft ? 1 : 0), rowY, colw, rowh);
			if (alternate) {
				if (alternateBase)
					option.features |= QStyleOptionViewItem::Alternate;
				else
					option.features &= ~QStyleOptionViewItem::Alternate;
			}
		d->drawCell(&painter, option, index);
	}
```
这一段代码，首先根据行号，列号以及父节点，得到索引，使用`d->drawCell(&painter, option, index);`绘制数据项，其实调用委托中的paint来绘制的
```bash
void QTableViewPrivate::drawCell(QPainter *painter, const QStyleOptionViewItem &option, const QModelIndex &index)
{
	Q_Q(QTableView);
	
	....
	
	q->itemDelegate(index)->paint(painter, opt, index);
}
```
在前面已经介绍过QTableView的默认委托为QStyledItemDelegate，所以`q->itemDelegate(index)->paint(painter, opt, index);`实际上是调用的是QStyledItemDelegate中的paint()，
```bash
void QStyledItemDelegate::paint(QPainter *painter,
	        const QStyleOptionViewItem &option, const QModelIndex &index) const
{
	....
	
	initStyleOption(&opt, index);
	
	....
}
```
然后通过QStyledItemDelegate中的initStyleOption函数，对数据项的各个角色的值进行初始化
```bash
QStylvoid QAbstractItemView::mouseDoubleClickEvent(QMouseEvent *event)
1938	{
1939	    Q_D(QAbstractItemView);edItemDelegate::initStyleOption(QStyleOptionViewItem *option,
	                                         const QModelIndex &index) const
{
	QVariant value = index.data(Qt::FontRole);
	
	....
	
	value = index.data(Qt::TextAlignmentRole);
	
	....
	
	value = index.data(Qt::ForegroundRole);
	
	....
	
	value = index.data(Qt::CheckStateRole);
	
	....
	
	value = index.data(Qt::DecorationRole);
	
	....
	
	value = index.data(Qt::DisplayRole);
	
	....
	
	option->backgroundBrush = qvariant_cast<QBrush>(index.data(Qt::BackgroundRole));

	....
}
```
通过index.data()，调用模型的data()方法。
这也就是为什么，自己定义一个模型类，重写data方法，使用qDebug简单做一个计数时，发现每一个数据项都调用了7次data方法，如果是2行3列的数据，则需要调用42次data方法。
最好能记住这7个角色的访问顺序，对你重写模型类的data方法的时候，或许会帮上你的忙，让你快速发现问题。Qt::DisplayRole文本显示，是最后才显示的。

#### 编辑数据 ####

当我们进行编辑数据时，双击单元格，修改数据，按下Tab，Backtab，Enter，Return或Esc时，数据就会被更新，这个过程其实是由委托来处理的。
鼠标双击时，视图的mouseDoubleClickEvent会捕捉到这个事件，
```bash
void QAbstractItemView::mouseDoubleClickEvent(QMouseEvent *event)
{
	Q_D(QAbstractItemView);
	
	....
	
	if ((event->button() == Qt::LeftButton) && !edit(persistent, DoubleClicked, event)
	        && !style()->styleHint(QStyle::SH_ItemView_ActivateItemOnSingleClick, 0, this))
	        emit activated(persistent);
```
然后在mouseDoubleClickEvent中会调用edit(persistent, DoubleClicked, event)，在edit函数中，会进一步去调用`d->openEditor(index, d->shouldForwardEvent(trigger, event) ? event : 0);`
```bash
bool QAbstractItemViewPrivate::openEditor(const QModelIndex &index, QEvent *event)
{
	Q_Q(QAbstractItemView);
	
	....
	
	QWidget *w = editor(buddy, options);
	
	....
}
```
重点就在editor函数中，在该函数中会使用委托来对数据进行更改
```bash
QWidget *QAbstractItemViewPrivate::editor(const QModelIndex &index,
	                                          const QStyleOptionViewItem &options)
{
	Q_Q(QAbstractItemView);
	QWidget *w = editorForIndex(index).widget.data();
		if (!w) {
			//得到委托，实际上就是QStyledItemDelegate
			QAbstractItemDelegate *delegate = delegateForIndex(index);
			if (!delegate)
				return 0;
			//创建一个编辑器，用来编辑数据
			w = delegate->createEditor(viewport, options, index);
			if (w) {
				//安装监听器，确保在使用Tab，Backtab，Enter，Return或Esc时可以将数据提交给模型修改数据
				w->installEventFilter(delegate);
				QObject::connect(w, SIGNAL(destroyed(QObject*)), q, SLOT(editorDestroyed(QObject*)));
				//使编辑器能够完整的显示出来
				delegate->updateEditorGeometry(w, options, index);
				//设置编辑器中的值
				delegate->setEditorData(w, index);
				addEditor(index, w, false);
				if (w->parent() == viewport)
					QWidget::setTabOrder(q, w);
	
				// Special cases for some editors containing QLineEdit
				QWidget *focusWidget = w;
				while (QWidget *fp = focusWidget->focusProxy())
					focusWidget = fp;
#ifndef QT_NO_LINEEDIT
				if (QLineEdit *le = qobject_cast<QLineEdit*>(focusWidget))
					le->selectAll();
#endif
#ifndef QT_NO_SPINBOX
				if (QSpinBox *sb = qobject_cast<QSpinBox*>(focusWidget))
					sb->selectAll();
				else if (QDoubleSpinBox *dsb = qobject_cast<QDoubleSpinBox*>(focusWidget))
					dsb->selectAll();
#endif
			}
	}

	return w;
}
```
注册监听器后，当有事件时，委托的eventFilter函数将会捕捉到事件，在事件处理函数中，当有键盘的Tab，Backtab，Enter，Return或Esc事件时，将会发送commitData信号和closeEditor信号，在QAbstractItemView::commitData()槽函数中使用setModelData(editor, d->model, index)，修改模型中的数据，其实就是调用了model的setData方法。在void QAbstractItemView::closeEditor()中删除编辑器和移除事件监听器。这两者之间的绑定位置在前面已经介绍过了，在创建视图的过程中就完整了它们之间信号和槽的绑定。

所以当我们自定义委托类时，就可以通过重写createEditor，updateEditorGeometry，setEditorData，setModelData方法，来满足我们的需求。

通过前面的介绍，明白了Model/View模型的工作过程，那么在编写需要满足自己要求的显示方式时，编写自定义的模型/委托时，将不会再感觉无从下手了。
