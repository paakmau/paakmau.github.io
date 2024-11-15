+++
title = "macOS 下使用 Makefile 基于 GLFW 配置 Dear ImGui"
date = 2020-03-31 23:47:45
slug = "202003312347"

[taxonomies]
tags = ["C++", "GLFW", "ImGui", "OpenGL"]
+++

用 Xcode 配置起来很简单，但是 `OpenGL.framework` 被弃用了，会爆不少警告（其实是因为我习惯了 VSC 改不过来）<br>

于是尝试写个 `Makefile` 搞一搞

<!-- more -->

## 安装 GLEW 和 GLFW

使用 `brew` 安装就行，它会把对应的库和头文放在在 `/usr/local/lib` 和 `/usr/local/include` 下

```sh
brew install glew
brew install glfw
```

## 获取 Dear ImGui 源码

我们可以直接把它的源码 Clone 下来

仓库地址在这里<br>
<https://github.com/ocornut/imgui>

然后其实看看 `examples` 里的例子抄抄就完事了（本文也是靠着它写的）

## 引入 Dear ImGui 库

首先建一个 `libs/imgui` 文件夹用于存放这个外部库

然后把项目源码根目录下的源文件跟头文件都拷贝进去<br>
还有 `examples` 下的 `imgui_impl_glfw.cpp`、`imgui_impl_glfw.h`、`imgui_impl_opengl3.cpp`、`imgui_impl_opengl3.h`

## `Makefile`

```makefile
EXE = imgui_test
SOURCES = main.cpp
SOURCES += imgui_impl_glfw.cpp imgui_impl_opengl3.cpp
SOURCES += imgui.cpp imgui_demo.cpp imgui_draw.cpp imgui_widgets.cpp
OBJS = $(addsuffix .o, $(basename $(notdir $(SOURCES))))

CXXFLAGS = -Ilibs/imgui/
CXXFLAGS += -g -Wall -Wformat
CXXFLAGS += -I/usr/local/include
CXXFLAGS += -std=c++17

LIBS = -framework OpenGL -framework Cocoa -framework IOKit -framework CoreVideo
LIBS += -L/usr/local/lib
LIBS += -lGLEW
LIBS += -lglfw

%.o:%.cpp
$(CXX) $(CXXFLAGS) -c -o $@ $<

%.o:libs/imgui/%.cpp
$(CXX) $(CXXFLAGS) -c -o $@ $<

all: $(EXE)
@echo Build complete

$(EXE): $(OBJS)
$(CXX) -o $@ $^ $(CXXFLAGS) $(LIBS)

clean:
rm -f $(EXE) $(OBJS)
```

## `main.cpp`

```cpp
#include "imgui.h"
#include "imgui_impl_glfw.h"
#include "imgui_impl_opengl3.h"

#include <GL/glew.h>
#include <GLFW/glfw3.h>

#include <cstdio>

static void
glfw_error_callback(int error, const char* description)
{
    fprintf(stderr, "Glfw Error %d: %s\n", error, description);
}

int main(int, char**)
{
    // Setup window
    glfwSetErrorCallback(glfw_error_callback);
    if (!glfwInit())
        return 1;

    // Decide GL+GLSL versions
    const char* glsl_version = "#version 150";
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE); // 3.2+ only
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE); // Required on Mac

    // Create window with graphics context
    GLFWwindow* window = glfwCreateWindow(1280, 720, "Dear ImGui GLFW+OpenGL3 example", NULL, NULL);
    if (window == NULL)
        return 1;
    glfwMakeContextCurrent(window);
    glfwSwapInterval(1); // Enable vsync

    // Initialize OpenGL loader
    if (glewInit() != GLEW_OK) {
        fprintf(stderr, "Failed to initialize OpenGL loader!\n");
        return 1;
    }

    // Setup Dear ImGui context
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    (void)io;

    // Setup Dear ImGui style
    ImGui::StyleColorsDark();

    // Setup Platform/Renderer bindings
    ImGui_ImplGlfw_InitForOpenGL(window, true);
    ImGui_ImplOpenGL3_Init(glsl_version);

    // Our state
    bool show_demo_window = true;
    bool show_another_window = false;
    ImVec4 clear_color = ImVec4(0.45f, 0.55f, 0.60f, 1.00f);

    // Main loop
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();

        // Start the Dear ImGui frame
        ImGui_ImplOpenGL3_NewFrame();
        ImGui_ImplGlfw_NewFrame();
        ImGui::NewFrame();

        // 1. Show the big demo window (Most of the sample code is in ImGui::ShowDemoWindow()! You can browse its code to learn more about Dear ImGui!).
        if (show_demo_window)
            ImGui::ShowDemoWindow(&show_demo_window);

        // 2. Show a simple window that we create ourselves. We use a Begin/End pair to created a named window.
        {
            static float f = 0.0f;
            static int counter = 0;

            ImGui::Begin("Hello, world!"); // Create a window called "Hello, world!" and append into it.

            ImGui::Text("This is some useful text."); // Display some text (you can use a format strings too)
            ImGui::Checkbox("Demo Window", &show_demo_window); // Edit bools storing our window open/close state
            ImGui::Checkbox("Another Window", &show_another_window);

            ImGui::SliderFloat("float", &f, 0.0f, 1.0f); // Edit 1 float using a slider from 0.0f to 1.0f
            ImGui::ColorEdit3("clear color", (float*)&clear_color); // Edit 3 floats representing a color

            if (ImGui::Button("Button")) // Buttons return true when clicked (most widgets return true when edited/activated)
                counter++;
            ImGui::SameLine();
            ImGui::Text("counter = %d", counter);

            ImGui::Text("Application average %.3f ms/frame (%.1f FPS)", 1000.0f / ImGui::GetIO().Framerate, ImGui::GetIO().Framerate);
            ImGui::End();
        }

        // 3. Show another simple window.
        if (show_another_window) {
            ImGui::Begin("Another Window", &show_another_window); // Pass a pointer to our bool variable (the window will have a closing button that will clear the bool when clicked)
            ImGui::Text("Hello from another window!");
            if (ImGui::Button("Close Me"))
                show_another_window = false;
            ImGui::End();
        }

        // Rendering
        ImGui::Render();
        int display_w, display_h;
        glfwGetFramebufferSize(window, &display_w, &display_h);
        glViewport(0, 0, display_w, display_h);
        glClearColor(clear_color.x, clear_color.y, clear_color.z, clear_color.w);
        glClear(GL_COLOR_BUFFER_BIT);
        ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());

        glfwSwapBuffers(window);
    }

    // Cleanup
    ImGui_ImplOpenGL3_Shutdown();
    ImGui_ImplGlfw_Shutdown();
    ImGui::DestroyContext();

    glfwDestroyWindow(window);
    glfwTerminate();

    return 0;
}
```
