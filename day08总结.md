## 总结
### day08 知识点
- 共享 VBO，双 VAO
- 两套 shander 场景
- 颜色逐分量相乘
- 物体颜色 = 官员颜色 * 物体反射颜色

### 场景结构
同一个立方体几何 + 两个 shader 程序 + 两个 VAO
被照射物体 shader：接受 objectColor 和 lightColor
光源 shader：直接输出白色（练习输出彩色）
共享 VBO 的多 VAO 写法：节省显存，减少重复上传
直觉：物体只能发射它颜色里已有的分量

### 物体 shander
```glsl
#version 330 core
out vec4 FragColor;

uniform vec3 objectColor;
uniform vec3 lightColor;

void main()
{
    FragColor = vec4(lightColor * objectColor, 1.0);
}
```

### 光源 shader
```glsl
#version 330 core
out vec4 FragColor;

void main()
{
    FragColor = vec4(1.0);
}
```

### 共享 VBO 的多 VAO 写法
```cpp
unsigned int VBO,cubeVAO,lightVAO;
glGenBuffers(1,&VBO);
glGenVertexArrays(1,&cubeVAO);
glGenVertexArrays(1,&lightVAO);

// 上传一次顶点数据
glBindBuffer(GL_ARRAY_BUFFER,VBO);
glBufferData(GL_ARRAY_BUFFER,sizeof(vertices),vertices,GL_STATIC_DRAW);

glBindVertexArray(cubeVAO);
glBindBuffer(GL_ARRAY_BUFFER,VBO);
glVertexAttribPointer(0,3,GL_FLOAT,GL_FALSE,3*sizeof(float),(void*)0);
glEnableVertexAttribArray(0);

glBindVertexArray(lightVAO);
glBindBuffer(GL_ARRAY_BUFFER,VBO);
glVertexAttribPointer(0,3,GL_FLOAT,GL_FALSE,3*sizeof(float),(void*)0);
glEnableVertexArrtibArray(0);

```

> VAO记录的是顶点属性指针的状态，包括当前绑定的 VBO。所以即使两个 VAO 共同用一个 VBO，只要分别在两个 VAO 上 glVertexAttribPointer 一遍，状态就分别保存下来了，draw 的时候 glBindVertexArray 即可切换。

### 渲染循环骨架
```cpp
glm::vec3 lightPos(1.2f,1.0f,2.0f);
glm::vec3 objectColor(1.0f,0.5f,0.31f);
glm::vec3 lightColor(1.0f,1.0f,1.0f);

while(!glfwWindowShouldClose(window)){
    // ... deltaTime / processInput / clear ...
    glm::mat4 view = camera.getViewMatrix();
    glm::mat4 projection = glm::perspective(glm::radians(camera.getZoom()),(float)W/H),0.1f,100.0f;

    // 画被照射的立方体
    lightingShader.use();
    lightingShader.setVec3("objectColor",objectColor);
    lightingShader.setVec3("lightColor",lightColor);
    lightingShader.setMat4("view",view);
    lightingShader.setMat4("projection", projection);

    glm::mat4 model(1.0f);
    lightingShader.setMat4("model", model);
    glBindVertexArray(cubeVAO);
    glDrawArrays(GL_TRIANGLES, 0 , 36);

    // 画光源立方体
    lightCubeShader.use();
    lightCubeShader.setMat4("view", view);
    lightCubeShader.setMat4("projection", projection);

    model = glm::mat4(1.0f);
    model = glm::translate(model,lightPos);
    model = glm::scale(model, glm::vec3(0.2f));
    lightCubeShader.setMat4("model", model);
    glBindVertexArray(lightVAO);
    glDrawArrays(GL_TRIANGLES,0,36);

    glfwSwapBuffers(window);
    glfwPollEvents();
}
```

## 回顾知识
- uniform ： 把 CPU 数据（纹理单元，mat4）传进 shader
- MVP：相机模型位置，相机视角矩阵，相机投影矩阵
- 