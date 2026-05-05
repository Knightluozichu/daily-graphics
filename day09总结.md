## 总结

### 公共模块

#### camera
> 维护一个 FPS 风格相机的状态，把键盘、鼠标输入转成位置和朝向变化，并输出 view matrix + fov（zoom）
##### 职责
- 存状态：相机在世界里的位置+朝向（用欧拉角和三个正交单位向量表示）
- 响应输入：键盘改位置，鼠标改朝向（yaw、pitch），滚轮改 fov（zoom）
- 输出数学：给主循环交两个东西--getViewMatrix（）（view 矩阵）和 getZoom（）（projection 用的 fov）
##### 成员
- 数据成员
  - position 世界空间的相机位置 （眼睛在哪）
  - front 相机的前方向，单位向量（眼睛看哪）
  - up 相机的上方向，单位向量 (lookAt 的第三个参数)
  - right 相机的右方向，单位向量 （用于A，D 平移）
  - worldUp 世界绝对的上方向（通常（010），是构造 right 的参考
  - yaw 偏航角，绕 y 轴左右转头（度）
  - pitch 俯仰角，绕 x 轴上下点头（度）
  - movementSpeed 键盘平移速度（单位、秒）
  - mouseSensitivity 鼠标灵敏度（度、像素）
  - zoom fov（度），滚轮改它来缩放
- 公开方法 （输入-做什么-返回）
  - Camera(position,up,yaw,pitch) 
    - 初始位姿
    - 设置成员，调用一次 updateCameraVectors()
    - 构造完成后状态自洽
  - getViewMatrix() const 
    - /
    - glm::lookAt(position, position+front,up)
    - 返回 mat4
  - getZoom() const 
    - /
    - /
    - 返回 fov
  - processKeyboard(dir,dt)
    - 方向枚举 + dt
    - position +- front * speed * dt(W/S) 或 +- right * speed *dt(A/D)
    - 改 position
  - processMouseMovement(dx, dy)
    - 屏幕 delta
    - yaw += dx * sens, pitch += dy * senes, 必要时夹pitch 到[-89,89],再 updateCameraVectors（）
    - 改 yaw、pitch + 三个轴
  - processMouseScroll（dy）
    - 滚轮 delta
    - zoom -= dy，夹到[1,45]
    - 改 zoom
- 内部方法
  - 输入：
    - 当前的 yaw、pitch、worldUp
  - 输出：
    - 刷新 front、right、up
    - front.x = cos(radians(yaw)) * cose(radians(pitch))
    - front.y = sin(radians(pitch))
    - front.z = sin(radians(yaw)) * cos(radians(pitch))
    - front = normalize(front)
    - right = normalize( cross(front, worldUp) )
    - up = normalize( cross(right, front) )
##### 标准流程
- 声明数据 + 默认常量
- 构造函数
- 写 updateCameraVectors
- 写 getViewMatrix 、 getZoom
- 写 三个 process
- 在 main 里接线

#### shader
> 把磁盘上的 GLSL 文件读进来，编译、链接成 program，并提供 setVec3 / setMat4 / setFloat 等设 uniform 的便捷方法
##### 职责
把磁盘上的.vs/.fs 文件读进内存-编译-链接成 program，再提供一组便捷的 sexXxx 把 CPP 数据塞到 shader 的 uniform 里，它是对 OpenGL5 个原生 API 的封装外壳：glShaderSource/glCompileShader/glAttachShader/glLinkProgram.glUniformXxx
它不做：
- 不画东西
- 不管 VAO、VBO
- 不管纹理绑定
##### 成员
- 字段
  - ID program 的 OpenGL 句柄（unsigned int）
  - >就这一个。所有其他东西（vertex shader、fragment shader 的 ID）都是构造函数里的临时变量，编译完链接完就 glDeleteShader 扔掉——它们已经被烧进 program 里了，不需要保留。
  - >关键认知：编译产物是"vertex shader 对象"和"fragment shader 对象"，但渲染时只用 program。两个 shader 对象 attach 到 program 后链接，链接完它们的使命就结束了。Shader 类只需要存 program ID。

- 公开方法（方法-输入-做什么-失败处理）
  - Shader(vertexPath,fragmentPath)
    - 两个路径字符串
    - 度两个文件-编译-链接-ID=program
    - 失败时 ID = 0 并打印诊断，不抛异常，调用方应在 use 前检查
  - use() const
    - /
    - glUseProgram(ID)
    - /
  - setBool
    - uniform 名+值
    - glUniform1i(loc,(int)v)
    - uniform 找不到时 loc = -1，glUniform1i(-1,...)是合法 no-op
  - > 关键约定：所有 setXxx 必须在 use 之后调用----glUniformXxx 影响的是当前绑定的 program。
  - 构造函数
    - 读文件
    - 拿 const char*
    - 编译两个 shader
    - 链接 program
    - 清理
- 私有方法
  - 错误检查辅助函数
    - > 把"取状态 → 失败时取日志"这段写成一个内部方法 checkCompileErrors(shader, type, name)：
    - type 用字符串 "VERTEX" / "FRAGMENT" / "PROGRAM" 区分
    - 当 type == "PROGRAM" 时用 glGetProgramiv + GL_LINK_STATUS + glGetProgramInfoLog
    - 否则用 glGetShaderiv + GL_COMPILE_STATUS + glGetShaderInfoLog
    - 错误信息里带上 name（路径）——以后多 shader 时一眼能看出是哪个文件出问题

##### 标准流程
- 头文件骨架
- 拆出读文件小函数
- 写构造函数主题
- 写 use
- 写 setXxx
- 写 checkCompileErrors


#### texture
> 加载图片文件 → OpenGL 纹理对象（一般用 stb_image）
##### 职责
把磁盘上的图片文件解码 → 上传到 GPU → 配置好采样参数，再提供 bind(slot) 把它挂到某个 texture unit 上。它是对 stb_image + 5 个 OpenGL 纹理 API（glGenTextures / glBindTexture / glTexImage2D / glGenerateMipmap / glTexParameteri）的封装。

它不做：

- 不绑到 sampler uniform（那是 main 通过 shader.setInt("texture1", 0) 做的事——把 sampler 指向 texture unit 0）
- 不画东西
- 不管 shader 里怎么采样

##### 成员
- 字段
  - ID         OpenGL 纹理对象句柄（unsigned int）
  - width      像素宽
  - height     像素高
  - channels   通道数（1=灰度, 3=RGB, 4=RGBA）
  - data       stb_image 解码后的 CPU 像素指针（unsigned char*）
- 公开方法
  - Texutre(path)
    - 文件路径
    - stb_image 加载 - 创建、绑定纹理 - 上传 - 生成 mipmap - 设过滤/wrap
    - 加载失败时 ID = 0，打印 stbi_failure_reason()
  - ~Texture()
    - /
    - stbi_image_free(data)（更好：构造里就释放，析构里 glDeleteTextures(1, &ID)）
    - /
  - bind(slot=0) const
    - texture unit 索引（0..N-1）
    - glActiveTexture(GL_TEXTURE0+slot) → glBindTexture(GL_TEXTURE_2D, ID)
    - /
  - unbind() const
    - /
    - glBindTexture(GL_TEXTURE_2D, 0)
    - /
  - getWidth/Height/Channels()
    - /
    - 返回成员
    - /
[CPU 解码]               [GPU 资源]                  [GPU 上传]               [采样参数]
路径 → 像素数组    →    glGenTextures + Bind   →   glTexImage2D + mipmap   →   wrap + filter
       ↓
   失败就报错


##### 标准流程
- 头文件骨架
- cpp 顶部
- 构造函数按上面 6 步写
- bind(slot) 两行
- unbind() 一行
- main 里的接线

### shader模块

#### lighting
一个 vs，一个 fs
##### 职责
- vs
  - 通过 mvp 矩阵计算出顶点在其次空间的位置gl_Position
  - 透传法线 Normal，FragPos 顶点在模型空间的位置，光源位置
- fs
  - 基础光照模型， 环境光ambient+漫反射 diffuse+高光 specular

#### lightCube
一个 vs，一个 fs
##### 职责
vs 光源位置，fs 返回颜色


### 基础光照模型的公式

#### ambient
ambientColor = ambientStrength * objectColor
#### diffuse
n = normalize(Normal)
l = normalize(LightPos - FragPos)
diff = max(dot(n,l),0.0)
diffuseColor = diff * lightColor

#### specular
n = normalize(Normal)
l = normalize(LightPos - FragPos)
v = normalize(viewPos - FragPos)
r = reflect(-l,n)
spec = pow(max(dot(v,r),0.0) , 32)
specularColor = spec * lightColor * specularStrength

### 遇到问题

#### shader编译报错
- 看报错，发现文件路径问题
- 错误的"形状"先看——一波报错挑出最早的根因，后面是连锁

#### 画面黑色
- 逐一调试 fs 的每一步输出，看预期画面在哪一步黑色了
- 没编译报错就用"画到屏幕"——把怀疑变量当颜色输出，每次只换一处，肉眼判断
- 多个观测互相印证——单看 norm 对、单看 lightDir 对，但 dot 错，就要怀疑两者之间的关系（方向、坐标空间）

#### amibientStrength 没有反应到cube身上？
- 法线没穿进去
- 因为cpu 没有给顶点布局的法线设置glVertexAttribPointer，以至于 gpu 不知道有法线
- glVertexAttribPointer和glEnableVertexAttribArray

#### diffuseColor为啥没生效？
- 字段名写错了

#### 排查diffuseColor全黑的流程步骤是啥？
- 就是刚说的 画面黑色

#### 光源移动，高光没有移动？
- 第一个问题，更新光源位置的时序
- 第二个问题，如何变换光源的位置
  -     model = glm::mat4(1.0f);
        model = glm::rotate(model,(float)glfwGetTime(), glm::vec3(0.0f,1.0f,0.0f));
        glm::vec4 p = model * glm::vec4(lightPos, 1.0f);
        actualLightPos = glm::vec3(p.x, p.y, p.z);
        model = glm::translate(model, lightPos);
        model = glm::scale(model, glm::vec3(0.2f));