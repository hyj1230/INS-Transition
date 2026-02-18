# INS-Transition

![](./output.webp)

## 简介

这是一个为 Python 对象提供 CSS 风格过渡（transition）功能的轻量级库。你可以定义多个**状态（state）**，每个状态为对象的属性指定目标值和过渡配置（ 持续时间、延迟、缓动函数），然后通过 `TransitionGroup` 在状态之间切换，实现属性的平滑过渡。其行为模仿了 Web 前端中 **CSS Transition** 的特性，包括**反转缩短（reverse shortening）**

## 安装

目前库以源码形式提供，你可以将文件放置在你的项目中直接导入：

```python
from INS-Transition import TransitionGroup, State, TransitionData
```

需要依赖 Python 3.6+。

## 核心概念

### TransitionData（过渡配置）

`TransitionData` 定义了过渡的持续时间、缓动函数和延迟。类似于 CSS 的 `transition-duration`、`transition-timing-function`、`transition-delay` 。

```python
data = TransitionData(
 transition_duration='0.5s', # 支持字符串如 '0.5s' 或数字（秒）
 transition_timing_function=ease, # 缓动函数
 transition_delay='0.1s' # 延迟时间
)
```

- `transition_duration` 以秒或毫秒为单位指定过渡动画所需的时间。默认值为 `'0s'`，表示不出现过渡动画。特别地，负数时间也会被存储为 `'0s'`。  
  其支持以下形式的参数：
  1. CSS 风格的字符串，如 `'100ms'`, `'.5s'`
  2. 一个整数或浮点数，以秒为单位
- `transition_timing_function` 设置如何计算受过渡效果影响的属性的中间值。
  其支持以下形式的参数：
  1. 代码形式，需要通过 `from INS-Transition import 需要的函数` 导入，具体包含：
     1. 预制的缓动函数 `linear`, `ease`, `ease_in`, `ease_in_out`, `ease_out`
     2. 自定义缓动函数，如 `CubicBezier(0.42, 0.0, 0.58, 1.0)`
  2. CSS 风格字符串形式，具体包含：
     1. 预制的缓动函数 `'linear'`, `'ease'`, `'ease-in'`, `'ease-in-out'`, `'ease-out'`
     2. 自定义缓动函数，如 `'cubic-bezier(.42, 0, .58, 1)'`
- `transition_delay` 规定了在过渡效果开始作用之前需要等待的时间。值的格式与 `transition_duration` 相同，表明动画过渡效果将在何时开始。取值为正时会延迟一段时间来响应过渡效果；取值为负时与 CSS 处理方式相同，这里不做介绍。

### Property（属性包装器）

`Property` 内部管理对象某个属性的过渡逻辑。通常你不需要直接创建它，而是通过 `TransitionGroup` 自动生成。

每个 `Property` 持有：
- `object_class`: 需要进行过渡操作的外部对象
- `name`: 属性的名称
- `value`：当前值（直接读写对象属性，通过 `getattr(object_class, name)` 以及 `setattr`）
- `after_change_value`：最终目标值
- `transition_data`：过渡配置
- `transition`：当前正在进行的过渡实例（内部）

### State（状态）

`State` 是一组属性目标值和过渡配置的集合。你可以通过 `State.create()` 快速创建。

对于状态中的每个属性，创建的格式如下：  
`属性名 = (目标值, TransitionData对象)`

```python
state_a = State.create(
 x=(100, TransitionData('1s')),
 y=(200, TransitionData('0.5s', linear))
)
```

状态之间可以继承：子状态若未定义某属性，会自动从父状态（通常是默认状态）获取。

每个 `State` 类内部不会存储状态的名称，状态的名称将在 `TransitionGroup` 中统一管理

### TransitionGroup（状态组）

`TransitionGroup` 是核心类，它持有一个对象和多个状态，负责在切换状态时更新所有属性的过渡。

你需要在每一帧调用 `update()` 方法来实时计算所有属性的过渡

切换状态需要使用 `set_state(字符串形式的状态名称)` 方法

```python
group = TransitionGroup(
 object_class=box,  # 要控制的对象
 default_state=default_state,  # 默认状态，其状态名称将被赋予为 'default'
 lerp_funcs={'x': my_lerp},  # 可选，为特定属性指定插值函数
 state1=state1, # 其他状态，格式为 状态名称 = State对象
 state2=state2
)
```

对于非数值类的属性，`lerp_funcs` 的指定是必要的。`lerp_funcs` 是一个字典，表示每个属性对应的插值函数（一般应当为线性插值）

自定义的插值函数要求符合以下格式：  
```python
def lerp(t, value1, value2):
    return new_value
```
其中，`t` 是表示进度的浮点数，决定了结果处于 `value1` 与 `value2` 之间的位置。特别地，0表示返回 `value1`，1表示返回 `value2`。

比如，这是内置的插值函数：
```python
def lerp(t: float, value1, value2):
    """线性插值函数"""
    return value1 + (value2 - value1) * t
```

另给出一个示例，这是用于插值 `pygame.Color` 的函数：
```python
def lerp_color(t: float, color1: pygame.Color, color2: pygame.Color):
    """颜色线性插值函数"""
    return color1.lerp(color2, t)
```

### Transition（运行时过渡实例）

`Transition` 是内部类，表示一个正在进行的过渡过程。用户一般无需直接操作它。

## 特性详解

### 类似 CSS 的过渡配置

- **持续时间**、**延迟** 支持字符串（如 `'0.3s'`）或数字（秒）。
- **缓动函数**：内置贝塞尔曲线缓动函数
- **多个属性可独立配置**：每个属性可以在不同状态中拥有自己的过渡参数。（若未配置，将自动从 `'default'` 状态继承）

### 反转缩短（Reverse Shortening）

这是 CSS 过渡的一个细节：当一个过渡正在进行时，如果目标值被改回过渡开始时的值，过渡不会重新开始，而是“反转”并缩短剩余时间。本库实现了这一行为。这一特性使得状态切换更加自然。

## API 参考

### `class TransitionData`
**初始化参数**：
- `transition_duration` (str/float)：持续时间，默认 `'0s'`。
- `transition_timing_function` (str/callable)：缓动函数，默认 `ease`。
- `transition_delay` (str/float)：延迟，默认 `'0s'`。

**属性**：
- `duration` (float)：持续时间（秒）。
- `timing_function` (callable)：缓动函数。
- `delay` (float)：延迟（秒）。
- `combine_duration` (float)：`delay + duration`。

### `class Property`
通常不直接实例化，由 `TransitionGroup` 自动创建。

**方法**：
- `update()`：更新属性值（基于当前过渡状态）。应在循环中调用。
- `value` (property)：获取/设置当前值（直接读写对象属性）。

### `class State`
**类方法**：
- `create(**kwargs)`：快速创建状态。参数格式为 `属性名=(目标值, TransitionData对象)`。
- `none_transition_data`：类属性，表示无过渡的默认配置。

**实例方法**：
- `add_property(property_name, target_value, transition_data=None)`：添加或更新属性。
- `inherit_state(father_state)`：从父状态继承未定义的属性。

### `class TransitionGroup`
**初始化参数**：
- `object_class`：要控制的对象。
- `default_state` (State)：默认状态。
- `lerp_funcs` (dict, optional)：为特定属性指定插值函数，格式 `{属性名: 函数}`。
- `**kwargs`：额外的状态，如 `state1=state_obj`。

**方法**：
- `add_state(name, state)`：添加一个命名状态。
- `add_states(**kwargs)`：批量添加状态。
- `update()`：计算所有属性的过渡。
- `set_state(new_state_name)`：切换到指定状态，更新每个属性的目标值和过渡配置。

**属性**：
- `current_state_name`：字符串，当前状态名。
- `property`：字典，`{属性名: Property对象}`。
- `states`：字典，`{状态名: State对象}`。

## 快速使用

按照以下四个步骤，你可以快速上手使用 INS-Transition 实现属性平滑过渡：

1. **选定需要应用过渡的属性**
   确定你想要实现动画效果的对象属性，例如位置（`x`, `y`）、尺寸（`width`, `height`）、颜色（`color`）等。这些属性将作为过渡的目标。

2. **设定 `'default'` 状态**
   创建一个 `State` 对象作为默认状态，为每个属性指定初始值。

   ```python
   default = State.create(
       x=(0, TransitionData('1s')),
       y=(0, TransitionData('1s'))
   )
   ```

   这类似于以下 CSS 代码：
   ```css
   .box {
       transform: translate(0px, 0px);
       transition: transform 1s ease;
   }
   ```

4. **设定其他状态**
   根据需要创建其他状态（如 `'hover'`、`'active'`）

   ```python
   hover = State.create(
       x=(100, TransitionData('0.5s', ease_out)),
       y=(200, TransitionData('0.5s', ease_out))
   )
   ```

   这类似于以下 CSS 代码：
   ```css
   .box:hover {
       transform: translate(100px, 200px);
       transition: transform 0.5s ease-out;
   }
   ```

5. **主循环中执行 `update()` 方法**
   将对象和状态绑定到 `TransitionGroup`，并在每一帧（如游戏循环）中调用其 `update()` 方法。切换状态时使用 `set_state()`，`update()` 会自动计算并更新对象的属性值。

   ```python
   group = TransitionGroup(my_object, default, hover=hover)

   while running:  # 主循环
       group.update()  # 每帧调用，更新属性
       # 渲染对象...
       if 条件满足:
           group.set_state('hover')  # 切换到 hover 状态
   ```

## 完整示例

下面是一个更完整的示例，你也可以查看 `slider_test.py` 这个更复杂的示例

```python
import pygame
import sys
from INS_Transition import State, TransitionGroup, TransitionData


pygame.init()
w, h = 800, 600
screen = pygame.display.set_mode((w, h), pygame.RESIZABLE)
clock = pygame.time.Clock()


class Button:
    def __init__(self):
        self.center = (400, 300)
        self.w, self.h = 200, 100
        
        self.state = False

        default_state = State.create(
            w = (self.w, TransitionData('1s')),
            h = (self.h, TransitionData('1s'))
        )
        focus_state = State.create(
            w = (self.w * 1.5, TransitionData('1s')),
            h = (self.h * 1.5, TransitionData('1s'))
        )
        
        self.transition_group = TransitionGroup(self, default_state, focus = focus_state)
        
    def get_rect(self):
        rect = pygame.Rect(0, 0, round(self.w), round(self.h))
        rect.center = self.center
        return rect

    def render(self, sf):
        pygame.draw.rect(sf, (255, 0, 0), self.get_rect(), border_radius=20)

    def handle_event(self, _event):
        if _event.type == pygame.MOUSEBUTTONDOWN and self.get_rect().collidepoint(_event.pos):
            self.state = not self.state
            self.transition_group.set_state('focus' if self.state else 'default')
                
    def update(self):
        self.transition_group.update()


button = Button()


while 1:
    pygame_events = pygame.event.get()
    for event in pygame_events:
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        button.handle_event(event)

    screen.fill('white')
    button.update()
    button.render(screen)

    pygame.display.flip()
    clock.tick(120)
```

