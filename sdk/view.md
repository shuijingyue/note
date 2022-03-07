## view的绘制

布局：计算view相对于父布局的位置和自己的尺寸，为绘制和触摸范围做支持
布局嵌套问题/自带的view和布局不满足需求，自定义view和layout
绘制：
触摸反馈：点击、滑动，获取触摸位置

### 流程

1. measure: 从根view递归调用子view的measure()方法，对其进行测量。计算自己的尺寸(期望尺寸)，父view根据子view的期望尺寸和内部实际情况来计算子vie
w的位置和实际尺寸
2. layout: 从根view递归调用子view的layout()方法，将子view的位置和实际尺寸传递给子view，子view将位置
和实际尺寸信息保存

#### measure流程

1. 父view在自己的onMeasure()方法，根据xml的配置信息对子view的要求和自身的可用空间，来获取子view
的具体尺寸要求
2. 子view在自己的onMeasure()中，根据父view的要求以及自己的特性计算自己的期望期望尺寸。如果子view
是`ViewGroup`，还会调用每个子view的measure方法()进行测量

子view为什么要根据父view的要求计算尺寸？

1. 父view中包含开发者的要求
2. 开:发者的要求可能不是具体的(例如: `match_parent`/`wrap_content`)

重写onMeasure
1. 计算size
2. resolveSize(size, measureSpec) / resolveSizeAndState(size, measureSpec)修正结果
3. setMeasuredDimension(width, height)

自定义ViewGroup必须重写layout

