# OGRE场景树模块

Ogre用场景树形式组织场景，而该模块的功能是将场景中所需要用到的对象存储起来，以节点、骨骼、八叉树等不同组织方式进行描述场景树。
场景结构图如下：

![](https://i.bmp.ovh/imgs/2021/03/9281a946f59503f8.png)

根节点的位置往往在场景的中心。每个节点对象都用数据成员保存相对于父节点的相对位移和相对旋转量（这一点很重要）。节点类提供节点的移动和旋转函数。这样当节点1发生移动和旋转的时候，其全部子节点及它们挂接的场景元素也将被动地移动和旋转。

引擎模块主要类介绍：

这是有关场景树的UML图。作为场景中可以渲染 object 的父类，场景树中的每个节点都是可渲染的，所以都是从 Renderable 中集成而来。同场景树相关的类主要是 Node、SceneNode。

![](https://i.bmp.ovh/imgs/2021/03/43cf420f51ae787d.png)

## Node类

一个虚类，定义了大部分操作和接口。Node为场景图的节点基类，定义公共属性。

- 有指向多个子节点的指针表
- 有指向多个父节点的指针表
- 增加、删除、修改节点操作
- 包围体和变换计算等操作
- Node派生类为各类场景节点，如可移动对象类、静态场景类、角色类等

继承图：

![](https://i.bmp.ovh/imgs/2021/03/1670d054742ef2db.png)

### 类型定义及成员变量

```c++
public:
    // 用以标记变换空间的枚举
    enum TransformSpace
    {
    	// 基于本地坐标系的变换
    	TS_LOCAL,
        // 基于父节点空间的变换
        TS_PARENT,
        // 基于世界坐标系的变换
        TS_WORLD
    };
	// 指向子节点的指针数组
    typedef std::vector<Node*> ChildNodeMap;
	// 子节点指针数组的迭代器
    typedef VectorIterator<ChildNodeMap> ChildNodeIterator;
	// 子节点指针数组的const迭代器
    typedef ConstVectorIterator<ChildNodeMap> ConstChildNodeIterator;

	class _OgreExport Listener
    {
    public:
        Listener() {}
        virtual ~Listener() {}
     	/* 
        	注意Listener是在新的继承关系出现时作用而不是方法改变时，所以即使可能有数个状态改变，
        	也只有一个Listener方法被调用，即当节点图完全更新之后
        */
        // 获取节点事件的回调
        virtual void nodeUpdated(const Node*) {}
        virtual void nodeDestroyed(const Node*) {}
        virtual void nodeAttached(const Node*) {}
        virtual void nodeDetached(const Node*) {}
    };
protected:
    // 指向父节点的指针
    Node* mParent;
    // 指向各个子节点的数组
    ChildNodeMap mChildren;

    typedef std::set<Node*> ChildUpdateSet;
    // 装载需要更新的子节点的集合，当有子节点过时而本身未过时的时候使用
    ChildUpdateSet mChildrenToUpdate;
    // 节点名
    String mName;

	// 表明需要从父节点获得更新
    mutable bool mNeedParentUpdate : 1;
    // 表明所有子节点需要更新
    bool mNeedChildUpdate : 1;
    // 表明父节点已经收到了更新请求
    bool mParentNotified : 1;
    // 表明该节点已经在排队等候更新
    bool mQueuedForUpdate : 1;
    // 标记该节点是否从父节点处继承方向属性
    bool mInheritOrientation : 1;
    // 标记该节点是否从父结点处继承缩放比例属性
    bool mInheritScale : 1;
    mutable bool mCachedTransformOutOfDate : 1;

    // 保存该节点相对父节点的方向
    Quaternion mOrientation;
    // 保存该结点相对父节点的位置
    Vector3 mPosition;
    // 保存该节点相对父节点的缩放比例
    Vector3 mScale;

    // 用4*4矩阵的形式保存转换矩阵
    mutable Affine3 mCachedTransform;

    // 仅内部可用的函数
    virtual void setParent(Node* parent);

        /** 综合计算方向
        @par
            在SceneManager或父节点调用_updateFromParent时更新，
            综合自身的及父节点的方向来更新数据
        */
    mutable Quaternion mDerivedOrientation;

        /** 综合计算位置
        @par
            在SceneManager或父节点调用_updateFromParent时更新，
            综合自身的及父节点的位置来更新数据
        */
    mutable Vector3 mDerivedPosition;

        /** 综合计算比例
        @par
            在SceneManager或父节点调用_updateFromParent时更新，
            综合自身的及父节点的缩放比例来更新数据
        */
    mutable Vector3 mDerivedScale;
	// 用于关键帧动画基础的位置
    Vector3 mInitialPosition;
    // 用于关键帧动画基础的方向
    Quaternion mInitialOrientation;
    // 用于关键帧动画基础的缩放比例
    Vector3 mInitialScale;

    // 有且仅有一个
    Listener* mListener;

    std::unique_ptr<DebugRenderable> mDebug;

    // 绑定用户对象
    UserObjectBindings mUserObjectBindings;

    typedef std::vector<Node*> QueuedUpdates;
    static QueuedUpdates msQueuedUpdates;
```

### 成员函数

#### 构造函数与析构函数

```c++
public:
    // 构造函数，只应该被父节点间接地调用
    Node();
    // 命名
    Node(const String& name);
    // 析构函数，声明为虚函数
    virtual ~Node(); 
    // 返回名字
    const String& getName(void) const { return mName; }
    // 返回父节点，若当前节点为根节点则返回NULL
    Node* getParent(void) const { return mParent; }
```

#### 位置相关函数

```C++
public:
        // 设置相对父节点的位置
        void setPosition(const Vector3& pos);
        // 重载
        void setPosition(Real x, Real y, Real z) { setPosition(Vector3(x, y, z)); }
        // 获取相对父节点的位置
        const Vector3 & getPosition(void) const { return mPosition; }

		/** 获取节点的初始位置 
        @remarks
            同时重置用于混合的累计动画比重
        */
        const Vector3& getInitialPosition(void) const { return mInitialPosition; }
        
        // 根据世界坐标，计算出该节点的本地坐标
        Vector3 convertWorldToLocalPosition( const Vector3 &worldPos );

        // 获取节点中某个点在世界坐标系下的坐标
        Vector3 convertLocalToWorldPosition( const Vector3 &localPos );

        // 给出世界坐标系，获取该节点的本地坐标系方向
        Vector3 convertWorldToLocalDirection( const Vector3 &worldDir, bool useScale );

        // 获取节点中某个点在世界坐标系下的方向
        Vector3 convertLocalToWorldDirection( const Vector3 &localDir, bool useScale );

        // 用四元数，通过世界坐标系获取本地坐标系下的方向
        Quaternion convertWorldToLocalOrientation( const Quaternion &worldOrientation );

        // 用四元数，获取该节点中某个点在世界坐标系下的方向
        Quaternion convertLocalToWorldOrientation( const Quaternion &localOrientation );
```

#### 变换相关函数

```c++
public:	
		// 返回表示节点方向的四元数
		const Quaternion & getOrientation() const { return mOrientation; }

        /** 通过四元数设置节点的方向
        @remarks
            方向不总是被继承的，继承与否取决于setInheritOrientation属性
        @par
            注意：旋转是围绕原点进行的
        */

		// 通过四元数设置方向
        void setOrientation( const Quaternion& q );
        // 重载
        void setOrientation( Real w, Real x, Real y, Real z);

        /** 重置节点的方向，即用世界坐标系作为物体坐标系
        @remarks
            方向不总是被继承的，继承与否取决于setInheritOrientation属性
        @par
            注意：旋转是围绕原点进行的
        */
        void resetOrientation(void);

        /** 设置用于该节点的缩放比例
        @remarks
            方向不总是被继承的，继承与否取决于setInheritScale属性
        @par
            注意：缩放是围绕原点进行的
        */
        void setScale(const Vector3& scale);
        // 重载
        void setScale(Real x, Real y, Real z) { setScale(Vector3(x, y, z)); }

        // 获取该节点的缩放信息
        const Vector3& getScale(void) const { return mScale; }

        // 设置该节点是否从父节点继承方向属性
        void setInheritOrientation(bool inherit);
        // 获取该节点是否从父节点继承方向属性
        bool getInheritOrientation(void) const { return mInheritOrientation; }

        // 设置该节点是否从父节点继承缩放属性
        void setInheritScale(bool inherit);
		// 获取该节点是否从父节点继承缩放属性
        bool getInheritScale(void) const { return mInheritScale; }

        // 缩放节点，结合当前的缩放值来重新计算缩放值
        void scale(const Vector3& scale);
        // 重载
        void scale(Real x, Real y, Real z);

        // 沿笛卡尔坐标系移动
        void translate(const Vector3& d, TransformSpace relativeTo = TS_PARENT);
        // 重载
        void translate(Real x, Real y, Real z, TransformSpace relativeTo = TS_PARENT)
        {
            translate(Vector3(x, y, z), relativeTo);
        }

        /** 沿任意坐标系移动
        @remarks
            该坐标系基于一组经典坐标系向量
        @param axes
            相对坐标系
        @param move
            在目标坐标系中的移动向量
        @param relativeTo
            这次变换发生的空间
        */
        void translate(const Matrix3& axes, const Vector3& move, TransformSpace relativeTo = TS_PARENT)
        {
            translate(axes * move, relativeTo);
        }
        // 重载
        void translate(const Matrix3& axes, Real x, Real y, Real z, TransformSpace relativeTo = TS_PARENT)
        {
            translate(axes, Vector3(x, y, z), relativeTo);
        }

		// 沿z轴旋转节点
        virtual void roll(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
        {
            rotate(Quaternion(angle, Vector3::UNIT_Z), relativeTo);
        }
        // 沿x轴旋转节点
        virtual void pitch(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
        {
            rotate(Quaternion(angle, Vector3::UNIT_X), relativeTo);
        }
        // 沿y轴旋转节点
        virtual void yaw(const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
        {
            rotate(Quaternion(angle, Vector3::UNIT_Y), relativeTo);
        }

        // 沿任意轴旋转节点，用相对坐标系计算
        void rotate(const Vector3& axis, const Radian& angle, TransformSpace relativeTo = TS_LOCAL)
        {
            rotate(Quaternion(angle, axis), relativeTo);
        }
		// 沿任意轴旋转节点，用四元数计算
        void rotate(const Quaternion& q, TransformSpace relativeTo = TS_LOCAL);

        // 获取一个矩阵，列是相对于父节点方向的轴
        Matrix3 getLocalAxes(void) const;

        /** 直接设置节点的最终世界坐标
        @remarks 
            一般还是建议调整节点的本地坐标
        */
        void _setDerivedPosition(const Vector3& pos);

        /** 直接设置节点的最终世界方向
        @remarks 
            一般还是建议调整节点的本地方向
        */
        void _setDerivedOrientation(const Quaternion& q);

        // 获取该节点在计算了所有父节点的继承之后的最终方向
        const Quaternion & _getDerivedOrientation(void) const;

        // 获取该节点在计算了所有父节点的继承之后的最终位置
        const Vector3 & _getDerivedPosition(void) const;

        // 获取该节点在计算了所有父节点的继承之后的最终缩放比例
        const Vector3 & _getDerivedScale(void) const;

        /** 直接获取该节点的总变换矩阵
        @remarks
            这个方法只应该被SceneManager得知变换已经更新完毕后调用
        */
        const Affine3& _getFullTransform(void) const;
```

#### 子节点相关函数

```c++
public:
		/** 创建一个未命名的节点作为该节点的子节点
        @param translate
            子节点相对于父节点的位移
        @param rotate
            子节点相对于父节点的旋转
        */
        virtual Node* createChild(
            const Vector3& translate = Vector3::ZERO, 
            const Quaternion& rotate = Quaternion::IDENTITY );

        /** 创建一个已命名节点作为该节点的子节点
        @remarks
            便于父节点在自己的子节点数组中找到目的子节点
        @param name 
        	名字
        @param translate
            子节点相对于父节点的位移
        @param rotate
            子节点相对于父节点的旋转
        */
        virtual Node* createChild(const String& name, const Vector3& translate = Vector3::ZERO, const Quaternion& rotate = Quaternion::IDENTITY);

        /** 将一个节点添加到当前节点的子节点数组中
        @param child 
        	要添加的子节点
        */
        void addChild(Node* child);

        /** 返回该节点的子节点数
        @尽量用getChildren()
        */
        uint16 numChildren(void) const { return static_cast< uint16 >( mChildren.size() ); }

        /** 根据下标获取目标子节点
        @remarks
            可以获取一个已命名子节点
        @尽量用getChildren()
        */
        Node* getChild(unsigned short index) const;

        // 通过子节点名字获取一个子节点
        Node* getChild(const String& name) const;

        // @尽量用getChildren()
        OGRE_DEPRECATED ChildNodeIterator getChildIterator(void);

        // @尽量用getChildren()
        OGRE_DEPRECATED ConstChildNodeIterator getChildIterator(void) const;

        // 返回当前节点的子节点数组
        const ChildNodeMap& getChildren() const { return mChildren; }

        /** 通过下标丢弃一个子节点
        @remarks
            解除绑定，但并不是删除该子节点
        */
        virtual Node* removeChild(unsigned short index);
        // 重载
        virtual Node* removeChild(Node* child);

        /** 通过名字丢弃一个子节点
        @remarks
            解除绑定，但并不是删除该子节点
        */
        virtual Node* removeChild(const String& name);
        // 解除所有子节点对该节点的绑定，但并不删除
        virtual void removeAllChildren(void);

		// 内部的用于创建子节点的方法，对每个子类都需要重写
        virtual Node* createChildImpl(void) = 0;
        // 内部的用于创建子节点的方法，对每个子类都需要重写
        virtual Node* createChildImpl(const String& name) = 0;
```

#### 其他函数

```c++
public:
        /** 为当前节点设置一个Listener
        @remarks
            由于大小和性能的原因，一个节点只能生效一个Listener
        */
        void setListener(Listener* listener) { mListener = listener; }
        
        // 获取当前节点的Listener
        Listener* getListener(void) const { return mListener; }
        

        /** 将当前的节点转换设置为“初始状态”，以作为一个基础用于关键帧动画
        @remarks
            除非你准备用这个节点制作动画，不然不要调用这个函数；
            若调用，则在这个节点一载入就调用
        @par
            如果没调用过这个函数，则reset将什么也不会发生
        */
        void setInitialState(void);

        // 回到初始状态
        void resetToInitialState(void);

		/** 更新节点的内部方法
        @note
            根据当前的变换更新这个节点及其子节点
            不要手动调用这个函数，除非你在写一个SceneManager的实现
        @param updateChildren
            如果为true，则更新将应用于所有子节点，否则将分别进行更新
        @param parentHasChanged
            表明父节点已经发生改变，子节点应该根据自己的状态来进行更新
        */
        virtual void _update(bool updateChildren, bool parentHasChanged);

		/** 在转换发生时被调用，重新计算节点
        @remarks
            不仅将节点标记为dirty，还将dirtiness告知父节点，以获取更新
        @param forceParentUpdate 
        	不管先前是否已经告知父节点，再次告知
        */
        virtual void needUpdate(bool forceParentUpdate = false);

        /** 子节点调用，告知父节点它需要更新
        @param child 
        	需要被更新的子节点
        @param forceParentUpdate  
        	不管先前是否已经告知父节点，再次告知
        */
        void requestUpdate(Node* child, bool forceParentUpdate = false);
        // 子节点告知父节点自己不再需要之前申请的更新
        void cancelUpdate(Node* child);

        // 尽量用DefaultDebugDrawer::drawAxes
        OGRE_DEPRECATED DebugRenderable* getDebugRenderable(Real scaling);

        /** 为一队needUpdate调用排队，以保证安全
        @remarks
            在场景图更新期间不能调用needUpDate，因为此时它的标记是不可靠的
            当你需要排列一队needUpdate请求的时候，使用这个方法
        */
        static void queueNeedUpdate(Node* n);
        // 执行这个队列
        static void processQueuedUpdates(void);


        /** 尽量别用UserObjectBindings::setUserAny，用getUserObjectBindings()
            为该物体设置任意用户参数
        @remarks
            这个方法允许用户将任意参数关联到该节点上
        */
        OGRE_DEPRECATED void setUserAny(const Any& anything) { getUserObjectBindings().setUserAny(anything); }
        /** 尽量别用UserObjectBindings::getUserAny，用getUserObjectBindings()
            检索节点的用户参数
        */
        OGRE_DEPRECATED const Any& getUserAny(void) const { return getUserObjectBindings().getUserAny(); }

        /** 
            返回与该类相关联的用户对象绑定的实例，一个或多个都可以
        @see UserObjectBindings::setUserAny.
        */
        UserObjectBindings& getUserObjectBindings() { return mUserObjectBindings; }

        /** 返回与该类相关联的用户对象绑定的实例，一个或多个都可以
        @see UserObjectBindings::setUserAny.
        */
        const UserObjectBindings& getUserObjectBindings() const { return mUserObjectBindings; }
    };
```

## SceneNode类

从代码中可以看出 SceneNode 类似于 Scene 中的一个 Cell，想象一下在 OctreeScene Manager 中的一个划分和 BSP 中的 cell。所以 SceneNode 可以认为是一个容器，可以存放 Movable Object。而存放在同一个 SceneNode 中的 object 应该有一定的共性。

继承图：

![](https://i.bmp.ovh/imgs/2021/03/fb804830b3406fd0.png)

### 类型定义及成员变量

```c++
public:
        typedef std::vector<MovableObject*> ObjectMap;
        typedef VectorIterator<ObjectMap> ObjectIterator;
        typedef ConstVectorIterator<ObjectMap> ConstObjectIterator;

    protected:
        ObjectMap mObjectsByName;

        /// SceneManager which created this node
        SceneManager* mCreator;

        /// World-Axis aligned bounding box, updated only through _update
        AxisAlignedBox mWorldAABB;

        void updateFromParentImpl(void) const;

        /** See Node */
        void setParent(Node* parent);

        /// Auto tracking target
        SceneNode* mAutoTrackTarget;
        /// Pointer to a Wire Bounding Box for this Node
        std::unique_ptr<WireBoundingBox> mWireBoundingBox;

        /** Index in the vector holding this node reference. Used for O(1) removals.

            It is the parent (or our creator) the one that sets this value, not ourselves. Do NOT modify
            it manually.
        */
        size_t mGlobalIndex;

        /// Tracking offset for fine tuning
        Vector3 mAutoTrackOffset;
        /// Local 'normal' direction vector
        Vector3 mAutoTrackLocalDirection;
        /// Fixed axis to yaw around
        Vector3 mYawFixedAxis;

        /// Whether to yaw around a fixed axis.
        bool mYawFixed : 1;
        /// Is this node a current part of the scene graph?
        bool mIsInSceneGraph : 1;

protected: // private in 1.13
        /// Flag that determines if the bounding box of the node should be displayed
        bool mShowBoundingBox : 1;
        bool mHideBoundingBox : 1;
```

### 成员函数

#### 构造函数与析构函数

```c++
public:
        /** Constructor, only to be called by the creator SceneManager.
        @remarks
            Creates a node with a generated name.
        */
        SceneNode(SceneManager* creator);
        /** Constructor, only to be called by the creator SceneManager.
        @remarks
            Creates a node with a specified name.
        */
        SceneNode(SceneManager* creator, const String& name);
        ~SceneNode();
```

#### Object相关的函数

AttachObject函数将场景对象的实例添加到此节点。场景对象可以包括实体对象，相机对象，灯光对象，粒子系统对象等。来自MovableObject的子类。还可以通过调用numAttachedObjects函数获取附加到此节点的对象数。可以通过索引或者对象的名称来获取附加的对象的指针。另外可以通过索引或名称或者指向对象的指针来分离一个附加的对象。最后，可以调用detachAllObjects把该节点上的所有对象分离。

```c++
public:
		/** Adds an instance of a scene object to this node.
        @remarks
            Scene objects can include Entity objects, Camera objects, Light objects, 
            ParticleSystem objects etc. Anything that subclasses from MovableObject.
        */
        virtual void attachObject(MovableObject* obj);

        /** Reports the number of objects attached to this node.
        @deprecated use getAttachedObjects()
        */
        unsigned short numAttachedObjects(void) const;

        /** Retrieves a pointer to an attached object.
        @remarks Retrieves by index, see alternate version to retrieve by name. The index
        of an object may change as other objects are added / removed.
        @deprecated use getAttachedObjects()
        */
        MovableObject* getAttachedObject(unsigned short index);

        /** Retrieves a pointer to an attached object.
        @remarks Retrieves by object name, see alternate version to retrieve by index.
        */
        MovableObject* getAttachedObject(const String& name);

        /** Detaches the indexed object from this scene node.
        @remarks
            Detaches by index, see the alternate version to detach by name. Object indexes
            may change as other objects are added / removed.
        */
        virtual MovableObject* detachObject(unsigned short index);
        /** Detaches an object by pointer. */
        virtual void detachObject(MovableObject* obj);

        /** Detaches the named object from this node and returns a pointer to it. */
        virtual MovableObject* detachObject(const String& name);

        /** Detaches all objects attached to this node.
        */
        virtual void detachAllObjects(void);

		/** Determines whether this node is in the scene graph, i.e.
            whether it's ultimate ancestor is the root scene node.
        */
        bool isInSceneGraph(void) const { return mIsInSceneGraph; }

        /** Notifies this SceneNode that it is the root scene node. 
        @remarks
            Only SceneManager should call this!
        */
        void _notifyRootNode(void) { mIsInSceneGraph = true; }
            

        /** Internal method to update the Node.
            @note
                Updates this scene node and any relevant children to incorporate transforms etc.
                Don't call this yourself unless you are writing a SceneManager implementation.
            @param
                updateChildren If true, the update cascades down to all children. Specify false if you wish to
                update children separately, e.g. because of a more selective SceneManager implementation.
            @param
                parentHasChanged This flag indicates that the parent transform has changed,
                    so the child should retrieve the parent's transform and combine it with its own
                    even if it hasn't changed itself.
        */
        void _update(bool updateChildren, bool parentHasChanged);

        /** Tells the SceneNode to update the world bound info it stores.
        */
        virtual void _updateBounds(void);

        /** Internal method which locates any visible objects attached to this node and adds them to the passed in queue.
            @remarks
                Should only be called by a SceneManager implementation, and only after the _updat method has been called to
                ensure transforms and world bounds are up to date.
                SceneManager implementations can choose to let the search cascade automatically, or choose to prevent this
                and select nodes themselves based on some other criteria.
            @param
                cam The active camera
            @param
                queue The SceneManager's rendering queue
            @param
                visibleBounds bounding information created on the fly containing all visible objects by the camera
            @param
                includeChildren If true, the call is cascaded down to all child nodes automatically.
            @param
                displayNodes If true, the nodes themselves are rendered as a set of 3 axes as well
                    as the objects being rendered. For debugging purposes.
            @param onlyShadowCasters
        */
        void _findVisibleObjects(Camera* cam, RenderQueue* queue,
            VisibleObjectsBoundsInfo* visibleBounds, 
            bool includeChildren = true, bool displayNodes = false, bool onlyShadowCasters = false);

        /** Gets the axis-aligned bounding box of this node (and hence all subnodes).
        @remarks
            Recommended only if you are extending a SceneManager, because the bounding box returned
            from this method is only up to date after the SceneManager has called _update.
        */
        const AxisAlignedBox& _getWorldAABB(void) const { return mWorldAABB; }

        /// @deprecated use getAttachedObjects()
        OGRE_DEPRECATED ObjectIterator getAttachedObjectIterator(void) {
            return ObjectIterator(mObjectsByName.begin(), mObjectsByName.end());
        }
        /// @deprecated use getAttachedObjects()
        OGRE_DEPRECATED ConstObjectIterator getAttachedObjectIterator(void) const {
            return ConstObjectIterator(mObjectsByName.begin(), mObjectsByName.end());
        }

        /** The MovableObjects attached to this node
         *
         * This is a much faster way to go through <B>all</B> the objects attached to the node than
         * using getAttachedObject.
         */
        const ObjectMap& getAttachedObjects() const {
            return mObjectsByName;
        }
```

#### 子节点相关函数

```c++
public:
		// 内部的用于创建子节点的方法，对每个子类都需要重写
        virtual Node* createChildImpl(void) = 0;
        // 内部的用于创建子节点的方法，对每个子类都需要重写
        virtual Node* createChildImpl(const String& name) = 0;

		/** Gets the creator of this scene node. 
        @remarks
            This method returns the SceneManager which created this node.
            This can be useful for destroying this node.
        */
        SceneManager* getCreator(void) const { return mCreator; }

        /** This method removes and destroys the named child and all of its children.
        @remarks
            Unlike removeChild, which removes a single named child from this
            node but does not destroy it, this method destroys the child
            and all of it's children. 
        @par
            Use this if you wish to recursively destroy a node as well as 
            detaching it from it's parent. Note that any objects attached to
            the nodes will be detached but will not themselves be destroyed.
        */
        void removeAndDestroyChild(const String& name);

        /// @overload
        void removeAndDestroyChild(unsigned short index);

        /// @overload
        void removeAndDestroyChild(SceneNode* child);


        /** Removes and destroys all children of this node.
        @remarks
            Use this to destroy all child nodes of this node and remove
            them from the scene graph. Note that all objects attached to this
            node will be detached but will not be destroyed.
        */
        void removeAndDestroyAllChildren(void);

		/** Creates an unnamed new SceneNode as a child of this node.
        @param
            translate Initial translation offset of child relative to parent
        @param
            rotate Initial rotation relative to parent
        */
        virtual SceneNode* createChildSceneNode(
            const Vector3& translate = Vector3::ZERO, 
            const Quaternion& rotate = Quaternion::IDENTITY );

        /** Creates a new named SceneNode as a child of this node.
        @remarks
            This creates a child node with a given name, which allows you to look the node up from 
            the parent which holds this collection of nodes.
            @param name name of the node
            @param
                translate Initial translation offset of child relative to parent
            @param
                rotate Initial rotation relative to parent
        */
        virtual SceneNode* createChildSceneNode(const String& name, const Vector3& translate = Vector3::ZERO, const Quaternion& rotate = Quaternion::IDENTITY);
```

#### 操作函数

```c++
public:
		/** Allows the showing of the node's bounding box.
        @remarks
            Use this to show or hide the bounding box of the node.
        */
        void showBoundingBox(bool bShow) { mShowBoundingBox = bShow; }

        /// @deprecated this function will disappear with 1.13
        OGRE_DEPRECATED void hideBoundingBox(bool bHide) { mHideBoundingBox = bHide; }

        /// @deprecated this function will disappear with 1.13
        OGRE_DEPRECATED void _addBoundingBoxToQueue(RenderQueue* queue);

        /** This allows scene managers to determine if the node's bounding box
            should be added to the rendering queue.
        @remarks
            Scene Managers that implement their own _findVisibleObjects will have to 
            check this flag and then use _addBoundingBoxToQueue to add the bounding box
            wireframe.
        */
        bool getShowBoundingBox() const { return mShowBoundingBox; }

		/** Allows retrieval of the nearest lights to the centre of this SceneNode.
        @remarks
            This method allows a list of lights, ordered by proximity to the centre
            of this SceneNode, to be retrieved. Can be useful when implementing
            MovableObject::queryLights and Renderable::getLights.
        @par
            Note that only lights could be affecting the frustum will take into
            account, which cached in scene manager.
        @see SceneManager::_getLightsAffectingFrustum
        @see SceneManager::_populateLightList
        @param destList List to be populated with ordered set of lights; will be
            cleared by this method before population.
        @param radius Parameter to specify lights intersecting a given radius of
            this SceneNode's centre.
        @param lightMask The mask with which to include / exclude lights
        */
        void findLights(LightList& destList, Real radius, uint32 lightMask = 0xFFFFFFFF) const;

        /** Tells the node whether to yaw around it's own local Y axis or a fixed axis of choice.
        @remarks
        This method allows you to change the yaw behaviour of the node - by default, it
        yaws around it's own local Y axis when told to yaw with TS_LOCAL, this makes it
        yaw around a fixed axis. 
        You only really need this when you're using auto tracking (see setAutoTracking,
        because when you're manually rotating a node you can specify the TransformSpace
        in which you wish to work anyway.
        @param
        useFixed If true, the axis passed in the second parameter will always be the yaw axis no
        matter what the node orientation. If false, the node returns to it's default behaviour.
        @param
        fixedAxis The axis to use if the first parameter is true.
        */
        void setFixedYawAxis( bool useFixed, const Vector3& fixedAxis = Vector3::UNIT_Y );

        /** Rotate the node around the Y-axis.
        */
        void yaw(const Radian& angle, TransformSpace relativeTo = TS_LOCAL);
        /** Sets the node's direction vector ie it's local -z.
        @remarks
        Note that the 'up' vector for the orientation will automatically be 
        recalculated based on the current 'up' vector (i.e. the roll will 
        remain the same). If you need more control, use setOrientation.
        @param x,y,z The components of the direction vector
        @param relativeTo The space in which this direction vector is expressed
        @param localDirectionVector The vector which normally describes the natural
        direction of the node, usually -Z
        */
        void setDirection(Real x, Real y, Real z,
            TransformSpace relativeTo = TS_LOCAL, 
            const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);

        /// @overload
        void setDirection(const Vector3& vec, TransformSpace relativeTo = TS_LOCAL,
            const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);
        /** Points the local -Z direction of this node at a point in space.
        @param targetPoint A vector specifying the look at point.
        @param relativeTo The space in which the point resides
        @param localDirectionVector The vector which normally describes the natural
        direction of the node, usually -Z
        */
        void lookAt( const Vector3& targetPoint, TransformSpace relativeTo,
            const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z);
        /** Enables / disables automatic tracking of another SceneNode.
        @remarks
        If you enable auto-tracking, this SceneNode will automatically rotate to
        point it's -Z at the target SceneNode every frame, no matter how 
        it or the other SceneNode move. Note that by default the -Z points at the 
        origin of the target SceneNode, if you want to tweak this, provide a 
        vector in the 'offset' parameter and the target point will be adjusted.
        @param enabled If true, tracking will be enabled and the next 
        parameter cannot be null. If false tracking will be disabled and the 
        current orientation will be maintained.
        @param target Pointer to the SceneNode to track. Make sure you don't
        delete this SceneNode before turning off tracking (e.g. SceneManager::clearScene will
        delete it so be careful of this). Can be null if and only if the enabled param is false.
        @param localDirectionVector The local vector considered to be the usual 'direction'
        of the node; normally the local -Z but can be another direction.
        @param offset If supplied, this is the target point in local space of the target node
        instead of the origin of the target node. Good for fine tuning the look at point.
        */
        void setAutoTracking(bool enabled, SceneNode* const target = 0,
            const Vector3& localDirectionVector = Vector3::NEGATIVE_UNIT_Z,
            const Vector3& offset = Vector3::ZERO);
        /** Get the auto tracking target for this node, if any. */
        SceneNode* getAutoTrackTarget(void) const { return mAutoTrackTarget; }
        /** Get the auto tracking offset for this node, if the node is auto tracking. */
        const Vector3& getAutoTrackOffset(void) const { return mAutoTrackOffset; }
        /** Get the auto tracking local direction for this node, if it is auto tracking. */
        const Vector3& getAutoTrackLocalDirection(void) const { return mAutoTrackLocalDirection; }
        /** Internal method used by OGRE to update auto-tracking cameras. */
        void _autoTrack(void);
        /** Gets the parent of this SceneNode. */
        SceneNode* getParentSceneNode(void) const;
        /** Makes all objects attached to this node become visible / invisible.
        @remarks    
            This is a shortcut to calling setVisible() on the objects attached
            to this node, and optionally to all objects attached to child
            nodes. 
        @param visible Whether the objects are to be made visible or invisible
        @param cascade If true, this setting cascades into child nodes too.
        */
        void setVisible(bool visible, bool cascade = true) const;
        /** Inverts the visibility of all objects attached to this node.
        @remarks    
        This is a shortcut to calling setVisible(!isVisible()) on the objects attached
        to this node, and optionally to all objects attached to child
        nodes. 
        @param cascade If true, this setting cascades into child nodes too.
        */
        void flipVisibility(bool cascade = true) const;

        /** Tells all objects attached to this node whether to display their
            debug information or not.
        @remarks    
            This is a shortcut to calling setDebugDisplayEnabled() on the objects attached
            to this node, and optionally to all objects attached to child
            nodes. 
        @param enabled Whether the objects are to display debug info or not
        @param cascade If true, this setting cascades into child nodes too.
        */
        void setDebugDisplayEnabled(bool enabled, bool cascade = true) const;

        /// @deprecated use DefaultDebugDrawer::drawAxes
        OGRE_DEPRECATED DebugRenderable* getDebugRenderable();
```

