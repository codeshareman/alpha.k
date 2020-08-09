---
title: ant-design 4.5.2源码 - form组件
date: 2020-08-09 19:07:16
tags: antd react
---

```javascript
// antd Form.tsx
const InternalForm: React.ForwardRefRenderFunction<unknown, FormProps> = (props, ref) => {}
React.useImperativeHandle(ref, () => wrapForm);
const Form = React.forwardRef<FormInstance, FormProps>(InternalForm);
```

1. Form 是用 [React.forwardRef](https://zh-hans.reactjs.org/docs/react-api.html#reactforwardref) 包裹的高阶组件, 可以从外部通过 ref 获取到 Form 实例
2. [useImperativeHandle](https://zh-hans.reactjs.org/docs/hooks-reference.html#useimperativehandle) 可以让你在使用 ref 时自定义暴露给父组件的实例值。在大多数情况下，应当避免使用 ref 这样的命令式代码。useImperativeHandle 应当与 forwardRef 一起使用

```javascript
// useForm.tsx
import { useRef, useMemo } from 'react';
import { useForm as useRcForm, FormInstance as RcFormInstance } from 'rc-field-form';
import scrollIntoView from 'scroll-into-view-if-needed';
import { ScrollOptions, NamePath, InternalNamePath } from '../interface';
import { toArray, getFieldId } from '../util';

export interface FormInstance extends RcFormInstance {
  scrollToField: (name: NamePath, options?: ScrollOptions) => void;
  /** This is an internal usage. Do not use in your prod */
  __INTERNAL__: {
    /** No! Do not use this in your code! */
    name?: string;
    /** No! Do not use this in your code! */
    itemRef: (name: InternalNamePath) => (node: React.ReactElement) => void;
  };
  getFieldInstance: (name: NamePath) => any;
}

function toNamePathStr(name: NamePath) {
  const namePath = toArray(name);
  return namePath.join('_');
}

export default function useForm(form?: FormInstance): [FormInstance] {
  const [rcForm] = useRcForm();
  const itemsRef = useRef<Record<string, React.ReactElement>>({});

  const wrapForm: FormInstance = useMemo(
    () =>
      form || {
        ...rcForm,
        __INTERNAL__: {
          itemRef: (name: InternalNamePath) => (node: React.ReactElement) => {
            const namePathStr = toNamePathStr(name);
            if (node) {
              itemsRef.current[namePathStr] = node;
            } else {
              delete itemsRef.current[namePathStr];
            }
          },
        },
        scrollToField: (name: string, options: ScrollOptions = {}) => {
          const namePath = toArray(name);
          const fieldId = getFieldId(namePath, wrapForm.__INTERNAL__.name);
          const node: HTMLElement | null = fieldId ? document.getElementById(fieldId) : null;

          if (node) {
            scrollIntoView(node, {
              scrollMode: 'if-needed',
              block: 'nearest',
              ...options,
            });
          }
        },
        getFieldInstance: (name: string) => {
          const namePathStr = toNamePathStr(name);
          return itemsRef.current[namePathStr];
        },
      },
    [form, rcForm],
  );

  return [wrapForm];
}
```
1. wrapForm 是通过useMemo缓存后的form实例，只有传入的form和rcForm变化时才会更新