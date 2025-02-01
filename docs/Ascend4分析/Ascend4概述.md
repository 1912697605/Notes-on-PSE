# ASCEND概述

来源：https://ascend4.org/ASCEND_overview

ASCEND是一个用于求解方程组的系统，主要面向工程师和科学家。它允许您通过从较简单的子模型构建复杂模型。使用ASCEND，您可以轻松地调整模型、检查其行为，并找出最佳求解方法。您可以轻松更改哪些变量是固定的，哪些是需要求解的，并且可以检查模型的求解过程。

## ASCEND中的简单模型示例
在ASCEND中，模型是用一种特殊的建模语言编写的，该语言结合了声明性变量和方程与过程性方法。以下简单模型计算水箱中水的质量：

```ascend
REQUIRE "atoms.a4l";

MODEL tank;
    V IS_A volume;
    A IS_A area;

    h IS_A distance;
    d IS_A distance;
    area_eqn: A = 1{PI} * d^2 / 4.;

    vol_eqn: V = A * h;
    rho IS_A mass_density;

    m IS_A mass;
    mass_eqn: m = rho*V;

METHODS
METHOD on_load;
    FIX d, h, rho;

    d := 50 {cm};
    h := 90 {cm};

    rho := 1000 {kg/m^3};
END on_load;
END tank;
```

该模型在模型的主部分声明变量和方程。然后，在一个名为`on_load`的方法中，模型指定变量`d`、`h`和`rho`将被“固定”，并提供它们的值。

该模型可以加载到ASCEND中，通常只需双击文件，然后点击“求解”按钮，得到以下结果：

![简单的水箱模型在ASCEND中加载并求解](https://ascend4.org/wiki/File:Tank_model_solved.png)

这是一个非常简单的模型：ASCEND可以做更多的事情。例如，无需编辑上述模型文件，可以使用GUI“固定”水箱的体积和直径，然后求解高度，只需几次点击，然后重新求解新的水箱体积为200升：

![您可以轻松“释放”和“固定”变量以求解不同的未知数集](https://ascend4.org/wiki/File:Tank_model_solved_for_height.png)

对于这样的模型，ASCEND可以快速显示它如何求解方程，使用简单的图形表示：

![对于小型模型，ASCEND可以生成求解过程的图形表示，有助于调试](https://ascend4.org/wiki/File:Tank_model_incidence.png)

像这样的简单模型可以是有用的，并且是检查简单方程组行为的有效方式。它们也可以作为快速的“计算器”：可以加载模型，输入已知变量，并快速查看求解结果。

ASCEND提供了一种方法来保存“观察”值表并绘制变量之间的关系：

![“观察者”选项卡允许计算和存储一组场景的结果，然后绘制图表](https://ascend4.org/wiki/File:Tank_model_observer.png)

请注意，ASCEND清晰透明地处理测量单位。在ASCEND中，您不必担心“单位是否正确”，因为它理解如何在不同单位之间进行转换，例如英寸、英尺、厘米、米、千米等长度单位，以及升、立方米等体积单位。

要了解更多关于ASCEND建模语言以及如何使用它构建模型的信息，请参阅《薄壁水箱教程》。

## 更复杂的模型
像这样的简单模型是有用的，但严重的问题，特别是在过程工程和机械工程中，通常涉及更复杂的交互，通常有多个子系统、相互连接的部件等。在这种情况下，ASCEND提供了许多面向对象的建模功能，允许声明更复杂的模型，例如这个简单的蒸汽动力站（朗肯循环）模拟：

```ascend
REQUIRE "johnpye/rankine.a4c";
MODEL power_station;

    (* 子模型 *)
    BO IS_A boiler_simple;
    TU IS_A turbine_simple;

    CO IS_A condenser_simple;
    PU IS_A pump_simple;

    (* 变量“别名”以便于使用 *)
    mdot ALIASES TU.mdot;

    (* 连接 *)
    BO.outlet, TU.inlet ARE_THE_SAME;

    TU.outlet, CO.inlet ARE_THE_SAME;
    CO.outlet, PU.inlet ARE_THE_SAME;

    PU.outlet, BO.inlet ARE_THE_SAME;

    (* 整体循环效率 *)
    eta IS_A fraction;

    eta * (BO.Qdot_fuel - PU.Wdot) = TU.Wdot;

METHODS
METHOD specify;
    RUN ClearAll;
    FIX PU.outlet.p, BO.outlet.T, PU.inlet.p, PU.inlet.h, TU.eta, PU.eta, BO.eta, mdot;

END specify;
METHOD values;
    mdot := 900 {t/d};

    PU.outlet.p := 2000 {psi};
    BO.outlet.T := 1460 {R};   BO.outlet.h := 3400 {kJ/kg};

    PU.inlet.p := 1 {psi};     PU.inlet.h := 69.73 {btu/lbm};

    TU.eta := 1.0;             PU.eta := 1.0;             BO.eta := 1.0;

END values;
METHOD on_load;
    RUN specify;
    RUN values;

END on_load;
END power_station;
```

该模型使用了几个子部件，如`pump_simple`等，这些子部件在另一个名为`rankine.a4c`的文件中定义，通过`REQUIRE`语句指向（完整代码请参见models/johnpye/rankine.a4c）。更多关于语法的文档可用，或者您可以查看我们在能源系统建模领域的其他模型。

使用这种模块化模型风格，可以构建和测试简单的组件模型，然后将它们“连接”成更复杂的模型。这是使用标准电子表格模型（工程师常用）难以实现的。有了标准组件模型库，您可以更快地开始解决面前的复杂问题。

请参阅《面向对象建模》以了解有助于编写更复杂模型的语言功能摘要。您可能还对我们在《使用电子表格建模》中的思考感兴趣。

## 求解器和积分器
上述示例中表示的基本工程计算是一个简单的“求解‘x’”类型的问题，ASCEND使用其QRSlv求解器来解决这些问题。对于这些问题，ASCEND对方程进行排序，然后使用牛顿法迭代求解系统中存在的任何联立方程组。然而，并非所有工程问题都是这样的。ASCEND还可以解决其他类型的问题：

- **优化问题**，其中必须通过改变一组输入变量来最小化目标值（参见CONOPT、IPOPT）
- **ODE和DAE问题**，其中需要研究时变系统的行为（参见LSODE、IDA）
- **条件模型**，其中适用的方程取决于模型中其他变量的值（参见CMSlv）

ASCEND GUI允许您加载模型，然后根据您的问题类型选择合适的“求解器”或“积分器”。ASCEND已经包含了几种求解器，并且API允许您编写新的求解器，而无需深入研究ASCEND的内部结构。

ASCEND可以向您展示求解器“看到”的模型结构。例如，使用QRSlv，简单的朗肯循环模型具有以下下三角关联矩阵结构，这表明不需要迭代求解：

![朗肯循环模型的关联矩阵结构](https://ascend4.org/wiki/File:Rankine-incidence1.png)

有关更多信息，请参阅《求解器和动态建模》。

## 热力学
过程工程师在建模时经常遇到困难的一个领域是在模型中融入流体的热力学性质。ASCEND提供了一些简单的热力学功能，这些功能已被证明足以用于化学工程中的蒸馏和闪蒸问题建模，并且还与高精度的IAPWS-IF97蒸汽表集成，用于电力工程应用。

有关更多信息，请参阅《使用ASCEND进行热力学建模》、《FPROPS》和《蒸汽性质》。

## 扩展ASCEND
ASCEND可以通过多种方式扩展，使用外部库。这些是ASCEND按需加载的代码片段。它们可以是：

- **外部方法**，对模型执行过程性操作
- **外部关系**，向模型添加新方程，例如复杂的热力学性质关系
- **外部求解器**，根据程序员的定义求解模型
- **外部积分器**，对模型进行积分（即求解系统的动态响应；求解初值问题）

外部库通常用C/C++编写，但我们也实现了支持使用Python编写外部方法。

## 脚本和自动化
ASCEND在几个不同层次上暴露其用户界面：C、C++和Python。这意味着，如果您有一个比ASCEND GUI能解决的更复杂的建模问题，您可以编写一个Python脚本，加载您的模型并以各种方式求解它：实际上，您可以“驱动”ASCEND，自动执行您想要的操作。

请参阅《使用Python脚本化ASCEND》。

## ASCEND的开发
ASCEND的功能足以对工程师、学生和科学家真正有用，并且有一个活跃的用户群体。它最早在1978年左右编写，自那时以来已经经历了多次重写。它的“编译器”（将输入文件转换为包含变量、子模型和方程的树结构）以及其求解器已经成为多个博士论文的主题，并且ASCEND中的许多功能已被纳入商业模拟程序中。然而，ASCEND的开发仍在朝着令人兴奋的新方向继续。我们正在努力添加新的求解器，并改进ASCEND对ODE和DAE求解的支持。

如果您有兴趣为ASCEND做出贡献，请参阅《开发活动》页面以及《学生项目列表》（GSOC）。