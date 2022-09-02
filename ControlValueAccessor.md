实现ControlValueAccessor接口，自定义表单元素。
主要是保持FormControl表单模型与组件数据同步
实现方法:
- writeValue(val)
val从表单模型或input => 组件类， 更新dom
- registerOnChange(fn)
fn在值发生变化时调用
- registerOnTouched(fn)
fn在失去焦点时调用
- setDisabledState
响应式表单，调用disable、enable方法时触发
或者input表单设置disabled属性时调用
