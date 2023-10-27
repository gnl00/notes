# React + TypeScript

## 组件值传递

### 父传子 props

```ts
// father component
import Son from '@/components/Test/Son'
export default function Father() {
  return (
    <div>
      <h2>This is father</h2>
      <Son num={100} />
    </div>
  )
}

// son component

interface ISonProps {
  num: number
  // ...
}
export default function Son(props: ISonProps) {
  return (
    <div>
      <h2>This is son</h2>
      <p>{props.num}</p>
    </div>
  )
}

```

…

---

###  子传父 useRef

```ts
// father
import { useEffect, useRef } from 'react'

export interface ISonRef {
  getValueFromSon: () => number
}

export default function Father() 
  const sonRef = useRef<ISonRef>(null) // 引用子组件
  useEffect(() => {
    if (sonRef && sonRef.current)
    console.log(sonRef.current.getValueFromSon())
  }, [])
  
  return (
    <div>
      <h2>This is father</h2>
      <Son num={100} ref={sonRef} />
    </div>
  )
}

// son
import React, { useImperativeHandle } from 'react'

interface ISonProps {
  num: number
  // ...
}

const Son = React.forwardRef((props: ISonProps, ref) => {
  // 子组件通过方法将值返回给父组件
  // useImperativeHandle is a React Hook that lets you customize the handle exposed as a ref.
  // https://react.dev/reference/react/useImperativeHandle
  useImperativeHandle(ref, () => ({
    getValueFromSon: (): number => 200
  }))

  return (
    <div>
      <h2>This is son</h2>
      <p>{props.num}</p>
    </div>
  )
})
export default Son
```

### 子调用父方法

父组件通过 props 将自己的方法转递给子组件执行

```tsx
// father
const fatherFunc = () => { console.log('father func') }

return (
  <div>
    <h2>This is father</h2>
    <Son invokeFunc={fatherFunc} />
  </div>
)
// son
interface ISonProps {
  invokeFunc: () => void
  // ...
}

const Son = React.forwardRef((props: ISonProps, ref) => {
  return (
    <div>
      <h2>This is son</h2>
      <button onClick={props.invokeFunc}>son click</button>
    </div>
  )
})
```

…