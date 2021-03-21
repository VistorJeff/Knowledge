# 场景树模块

Ogre 中使用场景树的方式组织场景，该模块的功能是将场景中所需要用到的对象存储起来，以节点、骨骼、八叉树等不同组织方式进行描述场景树。场景树的结构如下：

![Ogre场景树结构图](https://ftp.bmp.ovh/imgs/2021/03/be75cce496805eb7.png)

根节点的位置往往在场景的中心。每个节点对象都用数据成员保存相对于父节点的相对位移和相对旋转量。节点类提供节点的移动和旋转函数。这样当节点 1 发生移动和旋转的时候，其全部子节点及它们挂接的场景元素也将被动地移动和旋转。

场景树模块的主要类如下：

![Ogre场景树模块主要类](https://ftp.bmp.ovh/imgs/2021/03/e04f2152fb54c741.png)

这是有关场景树的 UML 图。作为场景中可以渲染 object 的父类，场景树中的每个节点都是可渲染的，所以都是从 Renderable 中集成而来。同场景树相关的类主要是 Node 和 SceneNode 。

## Node 类

Node 类是一个虚类，定义了大部分操作和接口，是场景图的节点基类，定义了公共属性：

- 指向多个子节点的指针表
- 指向多个父节点的指针表
- 增加、删除、修改节点操作
- 包围体和变换计算等操作

Node 派生类为各类场景节点，如可移动对象类、静态场景类、角色类等。

![Ogre-Node类继承图](https://ftp.bmp.ovh/imgs/2021/03/ec369004ccc53598.png)

### Node 类的类型定义和成员变量

```c++
public:
	// 标记变换空间的枚举类型变量
	enum TransformSpace
    {
        TS_LOCAL,   // 基于本地坐标系的变换
        TS_PARENT,  // 基于父节点空间的变换
        TS_WORLD    // 基于世界坐标系的变换
    };
	// 指向多个子节点的指针数组
	typedef std::vector<Node*> ChildNodeMap;
	// 访问子节点的迭代器
	typedef VectorIterator<ChildNodeMap> ChildNodeIterator;
	// 访问子节点的 const 迭代器
	typedef ConstVectorIterator<ChildNodeMap> ConstChildNodeIterator;
```

### Node 类的成员函数

#### 构造函数和析构函数

```c++
public:
	// 默认构造函数，初始化节点变量
	Node();
	// 传入一个 string 类型的节点名称，初始化节点变量
	Node(const String& name);
	// 析构函数是虚函数
	virtual ~Node();
```

#### 与位置和变换有关的函数

```c++
public:
	// 返回表示节点方向的四元数
	const Quaternion& getOrientation() const { return mOrientation; }
	// 通过四元数设置节点的方向
	// 与其它变换不同，方向并不总是由子节点继承，子节点是否继承方向取决于子节点的 setInheritOrientation 选项
	// 注意，旋转是围绕节点的原点定向的
	void setOrientation(const Quaternion& q);
	void setOrientation(Real w, Real x, Real y, Real z);
	void resetOrientation(void);
	// 设置是否继承父节点的方向
	void setInheritOrientation(bool inherit);
	// 返回该子节点是否继承了父节点的方向
	bool getInheritOrientation(void) const { return mInheritOrientation; }

	// 设置节点相对于其父节点的位置
	void setPosition(const Vector3& pos);
	void setPosition(Real x, Real y, Real z) { setPosition(Vector3(x, y, z)); }
	// 获得节点相对于其父节点的位置
	const Vector3& getPosition(void) const { return mPosition; }

	// 设置应用于此节点的缩放比例
	// 与其它变换不同，缩放比例并不总是由子节点继承，子节点是否继承方向取决于子节点的 setInheritScale 选项
	// 注意，像旋转一样，缩放比例也围绕节点的原点定向
	void setScale(const Vector3& scale);
	void setScale(Real x, Real y, Real z) { setScale(Vector3(x, y, z)); }
	// 获得节点的缩放比例
	const Vector3& getScale(void) const { return mScale; }
	// 设置是否继承父节点的缩放比例
	void setInheritScale(bool inherit);
	// 返回该子节点是否继承了父节点的缩放比例
	bool getInheritScale(void) const { return mInheritScale; }
	// 缩放节点，将其当前缩放比例与传入的缩放因子相结合
	void scale(const Vector3& scale);
	void scale(Real x, Real y, Real z);

	// 沿笛卡尔坐标轴移动节点
	void translate(const Vector3& d, TransformSpace relativeTo = TS_PARENT);
	void translate(Real x, Real y, Real z, TransformSpace relativeTo = TS_PARENT)
    {
        translate(Vector3(x, y, z), relativeTo);
    }
	// 沿任意轴移动节点
	void translate(const Matrix3& axes, const Vector3& move, TransformSpace relativeTo = TS_PARENT)
    {
        translate(axes * move, relativeTo);
    }
	void translate(const Matrix3& axes, Real x, Real y, Real z, TransformSpace relativeTo = TS_PARENT)
    {
        translate(axes, Vector3(x, y, z), relativeTo);
    }
	// 沿 z 轴旋转节点
	virtual void roll(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
    {
        rotate(Quaternion(angle, Vector3::UNIT_Z), relativeTo);
    }
	// 沿 x 轴旋转节点
	virtual void pitch(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
    {
        rotate(Quaternion(angle, Vector3::UNIT_X), relativeTo);
    }
	// 沿 y 轴旋转节点
	virtual void yaw(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
    {
        rotate(Quaternion(angle, Vector3::UNIT_Y), relativeTo);
    }
	// 沿任意轴旋转节点
	void rotate(const Vector3& axis, const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
    {
        rotate(Quaternion(angle, axis), relativeTo);
    }
	// 沿任意轴用四元数旋转节点
	void rotate(const Quaternion& q, TransformSpace relativeTo = TS_LOCAL);

	// 获取一个矩阵，矩阵的列是相对于父节点方向的轴
	Matrix3 getLocalAxes(void) const;

	// 获取从所有父节点派生的方向
	const Quaternion& _getDerivedOrientation(void) const;
	// 获取从所有父节点派生的位置
	const Vector3& _getDerivedPosition(void) const;
	// 获取从所有父节点派生的缩放比例
	const Vector3& _getDerivedScale(void) const;
	// 获取这个节点的整个变换矩阵
	// 应该只被 SceneManager 调用，确保在调用前派生的变换已经更新
	const Affine3& _getFullTransform(void) const;
```

#### 与子节点有关的函数

```c++
public:
	// 创建一个无名的子节点
	virtual Node* createChild(
        const Vector3& translate = Vector3::ZERO,
        const Quaternion& rotate = Quaternion::IDENTITY);
	// 创建一个有名的子节点
	virtual Node* createChild(
        const String& name,
        const Vector3& translate = Vector3::ZERO,
        const Quaternion& rotate = Quaternion::IDENTITY);
	// 添加一个子节点，如果这个节点已经连接到其它节点上了，就要先断开它们
	void addChild(Node* child);
	// 统计子节点的个数
	uint16 numChildren(void) const { return static_cast<uint16>(mChildren.size()); }
	// 获取一个指向子节点的指针，不推荐使用，属于一个备用的函数
	Node* getChild(unsigned short index) const;
	// 获取一个指向特定名字的节点的指针
	Node* getChild(const String& name) const;
	// 获取子节点列表
	const ChildNodeMap& getChildren() const { return mChildren; }
	// 与特定的子节点断开连接，注意不是删除子节点，而只是断开连接
	virtual Node* removeChild(unsigned short index);
	virtual Node* removeChild(Node* child);
	virtual Node* removeChild(const String& name);
	virtual void removeAllChildren(void);
```

