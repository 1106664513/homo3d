# sq更新优化：
1. openVDB本地复制编译
2. 一个自定义的一键编译脚本（需要先确定环境）
3. 命令行参数增加，传入目标值TargetValue（基于不同的优化目标）
   各向异性系数TargetA现在可以设定为约束，而不是固定的1。
5. `obj`命令更新：
 - Cubic X方向杨氏模量 $$E_x = \frac{1}{S_{11}}$$
 - Cubic 体对角线方向杨氏模量
 - G 目标值支持
# 一个面向大规模逆向均匀化问题的高性能、易用、开源 GPU 求解器

![image-20241111133209991](https://s2.loli.net/2024/11/11/jpC3TzYMWXArNIB.png)

本项目旨在提供一个高效的代码框架，用于求解**逆向均匀化问题**，从而设计微结构（microstructure）。

---

### 依赖项

* OpenVDB  
* CUDA 11  
* gflags  
* glm  
* Eigen3  

除 CUDA 和编译器外，其余依赖项已打包到一个 [conda](https://docs.conda.io/en/latest/miniconda.html) 环境中。你可以通过以下命令创建该环境：

```bash
conda env create -f environment.yml
```

然后激活该环境：

```bash
conda activate homo3d
```

---

### 编译

安装好依赖后，可使用 CMake 编译代码：

```shell
mkdir build
cd build
cmake ..
make -j4
```

如果已激活 conda 环境，`cmake` 会自动从该环境中查找所需依赖。

---

### 使用方法

#### 命令行参数

* `-reso`：离散化域的分辨率，例如 `-reso 128` 表示一个 $128 \times 128 \times 128$ 的计算域。默认值为 128。  
* `-obj`：要优化的目标函数，可选值包括 `bulk`（体模量）、`shear`（剪切模量）、`npr`（泊松比）和 `custom`（自定义目标）。默认为 `bulk`。  
* `-init`：密度场的初始化方法。常用且默认选项为 `randc`，即使用一组三角函数基进行初始化。也可设为 `manual`，从 OpenVDB 文件读取初始密度场。  
* `-sym`：结构的对称性约束，仅支持 `reflect3`、`reflect6` 和 `rotate3`。默认为 `reflect6`。  
* `-vol`：材料体积比，取值范围为 $(0,1)$，默认为 `0.3`。  
* `-E`：基体材料的杨氏模量。默认为 `1e1`（推荐值；不合适的值可能因 Fp16 表示范围有限而导致数值问题。后续可对弹性矩阵进行重缩放）。  
* `-mu`：基体材料的泊松比。默认为 `0.3`。  
* `-prefix`：输出路径（需以 `/` 结尾）。  
* `-in`：由其他参数决定的输入变量，例如当 `-init` 为 `manual` 时，此处应为 OpenVDB 文件路径。  
* `-N`：最大迭代次数，默认为 `300`。  
* `-relthres`：有限元方程求解的相对残差容差，默认为 `1e-2`。（注意：`master` 分支在容差小于 `1e-5` 时可能表现不佳，通常默认值已足够获得满意结果。）

---

#### 示例

优化体模量（bulk modulus）：

```shell
./homo3d -reso 128 -obj bulk -init randc -sym reflect6 -vol 0.3 -mu 0.3
```

优化完成后，优化后的密度场将以 OpenVDB 格式保存在 `<prefix>/rho` 文件中。

可使用第三方软件（如 Rhino 配合 Grasshopper 插件 [Dendro](https://www.food4rhino.com/en/app/dendro) 或 Blender）提取实体部分。

优化后的弹性张量矩阵以二进制格式保存在 `<prefix>/C` 中，包含 36 个单精度浮点数。

---

#### 自定义目标函数

若需优化自定义目标，请使用 `-obj custom` 选项，并在 `Framework.cu` 文件中添加你的目标函数和优化流程。我们已提供若干示例：

```cpp
void example_opti_bulk(cfg::HomoConfig config) {
    // ...
}
void example_opti_npr(cfg::HomoConfig config) {
    // ...
}
void example_yours(cfg::HomoConfig config) {
	// 在此处添加你自己的优化逻辑……
}
void runCustom(cfg::HomoConfig config) {
	//example_opti_bulk(config);
	//example_opti_npr(config);
	example_yours(config); // 取消此行注释
}
```

---

### 版本说明

如果你更关注**计算精度**而非性能，请切换到 `mix-fp64` 分支，并使用更小的有限元方程残差容差：

```bash
./homo3d -reso 128 -vol 0.1 -relthres 1e-6  # 将容差设为 1e-6
```

其他分支如 `mix-fp64fp32` 采用混合精度策略，可显著降低显存占用。

---

### 引用

若你在学术研究中使用本项目，请引用以下文献：

```
@ARTICLE{Zhang2023-ti,
  title    = "An optimized, easy-to-use, open-source {GPU} solver for
              large-scale inverse homogenization problems",
  author   = "Zhang, Di and Zhai, Xiaoya and Liu, Ligang and Fu, Xiao-Ming",
  journal  = "Structural and Multidisciplinary Optimization",
  volume   =  66,
  pages    =  "Article 207",
  month    =  sep,
  year     =  2023
}
```
