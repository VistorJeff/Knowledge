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
- 增加、删除、修改节点操作
- 包围体和变换计算等操作

Node 派生类为各类场景节点，如可移动对象类、静态场景类、角色类等。

![Ogre-Node类继承图](https://ftp.bmp.ovh/imgs/2021/03/ec369004ccc53598.png)



### Node 类的成员变量

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

	// Listener 获取节点事件的回调
	class _OgreExport Listener
    {
    public:
        Listener() {}
        virtual ~Listener() {}
        virtual void nodeUpdated(const Node*) {}
        virtual void nodeDestroyed(const Node*) {}
        virtual void nodeAttached(const Node*) {}
        virtual void nodeDetached(const Node*) {}
    };

protected:
	// 指向父节点的指针
	Node* mParent;
	// 指向多个子节点的指针数组
	ChildNodeMap mChildren;

	typedef std::set<Node*> ChildUpdateSet;
	// 需要更新的子节点列表，当自己没过时但是子节点过时时使用
	ChildUpdateSet mChildrenToUpdate;
	// 节点的名字
	String mName;

	// 来自父节点的私有变换已经过时
    mutable bool mNeedParentUpdate : 1;
    // 全部子节点都需要更新
    bool mNeedChildUpdate : 1;
    // 父节点已经收到更新请求
    bool mParentNotified : 1;
    // 已经在更新队列中等待
    bool mQueuedForUpdate : 1;
    // 是否从父节点中继承方向
    bool mInheritOrientation : 1;
    // 是否从父节点中继承缩放比例
    bool mInheritScale : 1;
    mutable bool mCachedTransformOutOfDate : 1;

	// 相对于父节点的方向
    Quaternion mOrientation;
    // 相对于父节点的位置
    Vector3 mPosition;
    // 该节点的缩放比例
    Vector3 mScale;

	// 缓存派生转换的 4x4 矩阵
	mutable Affine3 mCachedTransform;

	// 结合局部变换和父节点变换的方向
	// 当 SceneManager 或父节点调用 _updateFromParent 的时候更新
	mutable Quaternion mDerivedOrientation;
	// 同上，结合局部变换和父节点变换的位置
	mutable Vector3 mDerivedPosition;
	// 同上，结合局部变换和父节点变换的缩放比例
	mutable Vector3 mDerivedScale;

	// 用于关键帧动画基础的位置
	Vector3 mInitialPosition;
	// 用于关键帧动画基础的方向
	Quaternion mInitialOrientation;
	// 用于关键帧动画基础的缩放比例
	Vector3 mInitialScale;

	Listener* mListener;

	// 绑定用户对象
	UserObjectBindings mUserObjectBindings;

	typedef std::vector<Node*> QueuedUpdates;
	static QueuedUpdates msQueuedUpdates;
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

	// 获取节点的初始位置
	const Vector3& getInitialPosition(void) const { return mInitialPosition; }
	// 获取节点的初始方向
	const Quaternion& getInitialOrientation(void) const { return mInitialOrientation; }
	// 获取节点的初始缩放比例
	const Vector3& getInitialScale(void) const { return mInitialScale; }
	// 辅助函数，获取平方的视图深度
	Real getSquaredViewDepth(const Camera* cam) const;

	// 获取给定世界空间位置相对于此节点的局部位置
	Vector3 convertWorldToLocalPosition(const Vector3 &worldPos);
	// 获取节点局部空间中某个点的世界位置，这对于不需要子节点的简单变换很有用
	Vector3 convertLocalToWorldPosition(const Vector3 &localPos);
	// 获取给定世界空间方向相对于此节点的局部方向
	Vector3 convertWorldToLocalDirection(const Vector3 &worldDir, bool useScale);
	Quaternion convertWorldToLocalOrientation(const Quaternion &worldOrientation);
	// 获取节点局部空间中某个点的世界方向，这对于不需要子节点的简单变换很有用
	Vector3 convertLocalToWorldDirection(const Vector3 &localDir, bool useScale);
	Quaternion convertLocalToWorldOrientation(const Quaternion &localOrientation);
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

protected:
	virtual Node* createChildImpl(void) = 0;
	virtual Node* createChildImpl(const String& name) = 0;
```



#### 与节点有关的函数

```c++
public:
	// 返回节点的名字
	const String& getName(void) const { return mName; }
	// 获取该节点的父节点
	Node* getParent(void) const { return mParent; }

	// 在需要对该节点进行重新计算的变换更改时被调用，告诉父节点该节点已被修改，下次更新它
	virtual void needUpdate(bool forceParentUpdate = false);
	// 由子节点调用，通知其父节点他们需要更新
	void requestUpdate(Node* child, bool forceParentUpdate = false);
	// 由子节点调用，通知其父节点他们不再需要更新
	void cancelUpdate(Node* child);
	// 把节点更新命令存到队列中
	// 在场景图更新的过程中不能调用 needUpdate()
	static void queueNeedUpdate(Node* n);
	// 处理队列中的调用
	static void processQueuedUpdates(void);

	// 将此节点的当前变换设置为“初始状态”，即该位置/方向/比例将用作关键帧动画中使用的增量值的基础
	// 除非计划为该节点设置动画，否则无需调用此方法
	// 如果您打算对其进行动画处理，请在将节点的基本状态（即所有关键帧所基于的状态）加载到节点后，调用此方法
	void setInitialState(void);
	void resetToInitialState(void);

protected:
	// 设置父节点
	virtual void setParent(Node* parent);
	// 触发节点以更新其组合变换
	void _updateFromParent(void) const;
	// 具体类对于 _updateFromParent 的实现
	virtual void updateFromParentImpl(void) const;
```



## SceneNode 类

SceneNode 类继承自 Node 类，可以认为是一个容器，可以存放 Movable Object 。

```c++
class _OgreExport SceneNode : public Node
{
    friend class SceneManager;
    // ...
};
```



### SceneNode 类的成员变量

```c++
public:
	// 类型定义
	typedef std::vector<MovableObject*> ObjectMap;
	typedef VectorIterator<ObjectMap> ObjectIterator;
	typedef ConstVectorIterator<ObjectMap> ConstObjectIterator;

protected:
	ObjectMap mObjectsByName;
	// 创建这个节点的 SceneManager
	SceneManager* mCreator;
	// 世界坐标系对齐的包围盒，只通过 _update 更新
	AxisAlignedBox mWorldAABB;

	// 自动追踪目标
	SceneNode* mAutoTrackTarget;
	// 指向这个节点的
	std::unique_ptr<WireBoundingBox> mWireBoundingBox;

	// 保存此节点在节点引用数组中的下标
	size_t mGlobalIndex;

	// 跟踪偏移量以进行微调
	Vector3 mAutoTrackOffset;
	// 局部 normal 方向向量
	Vector3 mAutoTrackLocalDirection;
	// 固定轴偏航
	Vector3 mYawFixedAxis;

	// 是否对固定轴偏航
	bool mYawFixed : 1;
	// 该节点是不是场景图中的一个节点
	bool mIsInSceneGraph : 1;
	// 确定节点的包围盒是否显示
	bool mShowBoundingBox : 1;
	bool mHideBoundingBox : 1;
```



### SceneNode 类的成员函数



#### 构造函数和析构函数

```c++
public:
	// 只由 SceneManager 调用
	SceneNode(SceneManager* creator);
	SceneNode(SceneManager* creator, const String& name);
	~SceneNode();
```



#### 与位置和变换有关的函数

```c++
public:
	// 告诉节点偏航是绕自己的 Y 轴还是选定一个固定的轴
	// 只在使用自动追踪的时候有必要使用这个函数，因为手动旋转的时候可以详细指明 TransformSpace
	void setFixedYawAxis(bool useFixed, const Vector3& fixedAxis = Vector3::UNIT_Y);
	// 绕 Y 轴旋转
	void yaw(const Radian& angle, TransformSpace relativeTo = TS_LOCAL);
	
	// 设置节点的方向向量，即局部的 -Z
	// 方向的 up 向量会基于当前 up 向量自动地重新计算，如果需要更多控制的话就使用 setOrientation
	void setDirection(Real x, Real y, Real z,
        TransformSpace relativeTo = TS_LOCAL,
        const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);
	void setDirection(const Vector3& vec, TransformSpace relativeTo = TS_LOCAL,
        const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);
	// 指出这个节点在空间中一个点的局部 -Z 方向
	void lookAt( const Vector3& targetPoint, TransformSpace relativeTo,
        const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);

	// 开启或禁用对另一个 SceneNode 的自动追踪
	// 如果开启了自动追踪，这个 SceneNode 在每一帧都会自动旋转到目标 SceneNode 的 -Z 上
	// target 指向要追踪的 SceneNode 的指针，确保这个节点没有被删除
	void setAutoTracking(bool enabled, SceneNode* const target = 0,
        const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z,
        const Vector3& offset = Vector3::ZERO);
	// 获取这个节点自动追踪的目标
	SceneNode* getAutoTrackTarget(void) const { return mAutoTrackTarget; }
	// 获取这个节点自动追踪的偏移量
	const Vector3& getAutoTrackOffset(void) const { return mAutoTrackOffset; }
	// 获取这个节点自动追踪的局部方向
	const Vector3& getAutoTrackLocalDirection(void) const { return mAutoTrackLocalDirection; }
	// OGRE 用来更新自动追踪摄像机的内部方法
	void _autoTrack(void);
```



#### 与子节点有关的函数

```c++
public:
	// 移除并销毁一个特定的子节点以及这个子节点的所有孩子节点
	// 与 removeChild 不同，removeChild 只移除特定的子节点但不销毁它
	void removeAndDestroyChild(const String& name);
	void removeAndDestroyChild(unsigned short index);
	void removeAndDestroyChild(SceneNode* child);
	// 移除并销毁这个节点的全部子节点
	void removeAndDestroyAllChildren(void);

	// 创建一个无名的 SceneNode 作为自己的子节点
	// translate 相对于父节点初始的变换
	// rotate 相对于父节点初始的旋转
	virtual SceneNode* createChildSceneNode(
        const Vector3& translate = Vector3::ZERO, 
        const Quaternion& rotate = Quaternion::IDENTITY );
	// 同上，创建一个有名的 SceneNode 作为自己的子节点
	virtual SceneNode* createChildSceneNode(
        const String& name,
        const Vector3& translate = Vector3::ZERO,
        const Quaternion& rotate = Quaternion::IDENTITY);

protected:
	Node* createChildImpl(void);
	Node* createChildImpl(const String& name);
```



#### 与节点有关的函数

```c++
public:
	// 往节点中添加一个场景对象的实例
	// 场景对象可以包括实体对象、摄像机对象、灯光对象等，任意 MovableObject 的子类
	virtual void attachObject(MovableObject* obj);
	// 连接到这个节点的对象个数
	unsigned short numAttachedObjects(void) const; // 弃用
	// 取出一个对象的指针
	MovableObject* getAttachedObject(unsigned short index); // 弃用
	MovableObject* getAttachedObject(const String& name);
	// 获取所有连接到节点的可移动对象
	const ObjectMap& getAttachedObjects() const {
        return mObjectsByName;
    }

	// 断开对象与节点的连接
	virtual MovableObject* detachObject(unsigned short index);
	virtual void detachObject(MovableObject* obj);
	virtual MovableObject* detachObject(const String& name);
	virtual void detachAllObjects(void);

	// 节点在不在场景树中
	bool isInSceneGraph(void) const { return mIsInSceneGraph; }

	// 通知这个节点它是场景树的根节点，只有 SceneManager 可以调用
	void _notifyRootNode(void) { mIsInSceneGraph = true; }

	// 更新节点的内部方法
	// updateChildren 为 true 的话就会更新所有子节点，如果想分别更新子节点就设置为 false
	// parentHasChanged 表明父节点的变换已经改变，子节点需要获取到新的变换并与自己的变换结合
	void _update(bool updateChildren, bool parentHasChanged);

	// 更新边界
	virtual void _updateBounds(void);

	// 定位任意连接到该节点的可视对象并添加到渲染队列中
	// 只能被 SceneManager 实现调用，且只能在 _update 被调用后确保变换和世界边界已经更新
	void _findVisibleObjects(Camera* cam, RenderQueue* queue,
        VisibleObjectsBoundsInfo* visibleBounds, 
        bool includeChildren = true, bool displayNodes = false, bool onlyShadowCasters = false);

	// 获取这个节点（包括所有子节点）的坐标系对齐的包围盒
	// 只在扩展一个 SceneManager 的时候推荐使用，因为该函数返回的包围盒只在 SceneManager 调用 _update 后更新
	const AxisAlignedBox& _getWorldAABB(void) const { return mWorldAABB; }

	// 获取这个场景节点的创建者
	// 对销毁节点很有用
	SceneManager* getCreator(void) const { return mCreator; }

	// 允许显示节点的包围盒
	void showBoundingBox(bool bShow) { mShowBoundingBox = bShow; }
	// SceneManager 实现自己的 _findVisibleObjects 时需要检查这个标志
	// 然后使用 _addBoundingBoxToQueue 添加包围盒线框
	bool getShowBoundingBox() const { return mShowBoundingBox; }

	// 允许检索离 SceneNode 中心最近的灯光
	void findLights(LightList& destList, Real radius, uint32 lightMask = 0xFFFFFFFF) const;

	// 获取父节点
	SceneNode* getParentSceneNode(void) const;

	// 设置连接到这个场景节点的所有对象可见或不可见
	// cascade 如果是 true 的话，这个设置会同时作用到子节点上
	void setVisible(bool visible, bool cascade = true) const;
	// 反转连接到这个场景节点上的所有对象的可见性
	void flipVisibility(bool cascade = true) const;

	// 设置连接到场景节点的全部对象是否显示调试信息
	void setDebugDisplayEnabled(bool enabled, bool cascade = true) const;

protected:
	void updateFromParentImpl(void) const;
	void setParent(Node* parent);
	// 设置节点是否在场景图中的内部方法
	virtual void setInSceneGraph(bool inGraph);
```



## Bone 类

Bone 类主要存放对象骨骼信息，其中大部分成员函数都在 Mesh 类中大量重写。

```c++
class _OgreExport Bone : public Node
{
    // ...
private:
    // 故意隐藏 createChild 方法的基本实现
    // 同时抑制了比如 'Ogre::Bone::createChild' hides overloaded virtual functions 警告
    using Node::createChild;
};
```



### Bone 类的成员变量

```c++
protected:
	// 指向创建者的指针，用于子节点的创建
	Skeleton* mCreator;

	// “绑定姿势”下骨骼的反向继承缩放比例
	Vector3 mBindDerivedInverseScale;
	// “绑定姿势”下骨骼的反向继承方向
	Quaternion mBindDerivedInverseOrientation;
	// “绑定姿势”下骨骼的反向继承位置
	Vector3 mBindDerivedInversePosition;

	// 骨骼的 handle
	unsigned short mHandle;

	// 手动控制
	bool mManuallyControlled;
```



### Bone 类的成员函数



#### 构造函数和析构函数

```c++
public:
	// 不要直接使用（使用 Bone::createChild 或 Skeleton::createBone）
	Bone(unsigned short handle, Skeleton* creator);
	Bone(const String& name, unsigned short handle, Skeleton* creator);
	~Bone();
```



#### 与子节点有关的函数

```c++
public:
	// 创建一个新的 Bone 类作为子节点
	// 这个子节点会继承当前节点的变换
	// handle 是一个在 Skeleton 中唯一的数字
	Bone* createChild(unsigned short handle, 
        const Vector3& translate = Vector3::ZERO, const Quaternion& rotate = Quaternion::IDENTITY);

protected:
	Node* createChildImpl(void);
	Node* createChildImpl(const String& name);
```



#### 与节点有关的函数

```c++
public:
	// 获取当前节点的 handle
	unsigned short getHandle(void) const;

	// 设置当前方向/位置为“绑定姿势”，即骨骼最初绑定到网格上的布局
	void setBindingPose(void);
	// 把骨骼的位置和方向重新设置为初始状态
	void reset(void);

	// 设置这个骨骼是否手动控制
	void setManuallyControlled(bool manuallyControlled);
	// 查看这个骨骼是否手动控制
	bool isManuallyControlled() const;

	// 从“绑定姿势”获取骨骼空间到当前位置的变换
	// 只能在内部使用
	void _getOffsetTransform(Affine3& m) const;

	// 获取反转的“绑定姿势”的缩放比例
    const Vector3& _getBindingPoseInverseScale(void) const
    { return mBindDerivedInverseScale; }
    // 获取反转的“绑定姿势”的位置
    const Vector3& _getBindingPoseInversePosition(void) const
    { return mBindDerivedInversePosition; }
    // 获取反转的“绑定姿势”的方向
    const Quaternion& _getBindingPoseInverseOrientation(void) const
    { return mBindDerivedInverseOrientation; }

	// 见 Node::needUpdate
	void needUpdate(bool forceParentUpdate = false);
```

