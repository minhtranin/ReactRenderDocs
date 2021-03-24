Translated from [https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/), author: [Mark Erikson](https://twitter.com/acemarke) (from Redux team)

# A (Mostly) Complete Guide to React Rendering Behavior

Bài viết cung cấp chi tiết về cách mà React render hoạt động, và việc sử dụng Context và Redux ảnh hưởng thế nào tới quá trình render của React.

## "Render" là gì

> Rendering is the process of React asking your components to describe what they want their section of the UI to look like, now, based on the current combination of props and state.

Render là 1 quá trình xử lí của React yêu cầu các components trả về mô tả các thành phần UI trong component đó, dựa trên sự kết hợp của props và state.

### Tổng quan quá trình Render

Trong quá trình render, React sẽ bắt đầu với root component tree và lặp dần xuống dưới các component con để tìm ra những component đã được đã đánh dấu là cần cập nhật. Với mỗi component được đánh dấu này, React sẽ chạy `classComponentInstance.render()` (đối với các class-component) hoặc là chạy `FunctionComponent()` (đối với các functional-component) để lấy được output của quá trình render.

Render output của 1 component thường được viết bằng JSX, trong quá trình build (compile), JSX sẽ được convert thành các hàm `React.createElement()`. `createElement` trả về React elements (hay còn được biết đến với tên "Virtual DOM"), dưới dạng plain JS Object - cung cấp mô tả về cấu trúc của UI Component. Ví dụ:

```javascript
// Đây là JSX:
return <SomeComponent a={42} b="testing">Text here</SomeComponent>

// Khi build xong sẽ được convert thành:
return React.createElement(SomeComponent, {a: 42, b: "testing"}, "Text Here")

// Và khi trình duyệt execute compiled code, nó sẽ tạo ra React element object như sau:
{type: SomeComponent, props: {a: 42, b: "testing"}, children: ["Text Here"]}
```

Sau khi thu thập đủ render output từ component tree (kết quả là 1 React element object), React sẽ so sánh (diff) virtual DOM mới và virtual DOM hiện tại, thu được một tập hợp các thay đổi thực sự cần được cập nhật vào real DOM, quá trình so sánh và tính toán này được gọi là "[reconciliation](https://reactjs.org/docs/reconciliation.html)".

React sau đó áp dụng tất cả các thay đổi đã được tính toán ở trên lên cây DOM thật trong một thứ tự đồng bộ (Render Phase và Commit Phases).

### Render Phase và Commit Phases

React team chia Rendering Process thành 2 pha (phase):

- "Render phase" bao gồm tất cả công việc của việc render components và tính toán các thay đổi cần apply
- "Commit phase" là quá trình áp dụng các thay đổi này vào DOM thật

Sau khi React cập nhật lại DOM thật trong Commit Phase, nó sau đó chạy đồng bộ các methods `componentDidMount` và `componentDidUpdate` của class-component, và `useLayoutEffect` hooks.

React sau đó đặt một khoản thời gian ngắn (timeout), sau khi hết timeout thì nó sẽ chạy tất cả các `useEffect` hooks. Bước này được gọi là "Passive Event" phase.

Bạn có thể xem visualization của các class lifecycle methods [tại đây](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/).

> In [React's upcoming "Concurrent Mode"](https://reactjs.org/docs/concurrent-mode-intro.html), it is able to pause the work in the rendering phase to allow the browser to process events. React will either resume, throw away, or recalculate that work later as appropriate. Once the render pass has been completed, React will still run the commit phase synchronously in one step.

Trọng tâm của phần này là hiểu rằng "rendering" không phải là "updating the DOM", một component có thể được render mà không thay đổi gì trên DOM thật. Khi React render component:

- Nếu component trả về render output giống với lần render trước đó, sẽ không có thay đổi nào cần được áp dụng (lên DOM thật) -> không commit gì cả.
- In Concurrent Mode, React might end up rendering a component multiple times, but throw away the render output each time if other updates invalidate the current work being done

## Làm thế nào React handle Renders?

### Queuing Renders

Sau khi lần render đầu tiên (initial) được hoàn thành, có một vài cách để kích hoạt React render trên một vài component (đánh dấu là component đó cần update và React sẽ thực hiện quá trình re-render sau đó):

- Class components:
    - `this.setState()`
    - `this.forceUpdate()`
- Functional components:
    - `useState` setters
    - `useReducer` dispatches
- Khác:
    - Gọi `ReactDOM.render(<App/>)` lại lần nữa, tương đương với việc gọi `forceUpdate()` tại component root.
    
### Render Behavior tiêu chuẩn

Có một điều quan trọng phải nhớ:

**React's default behavior là khi có một component cha render, React sẽ lặp đệ quy và render tất cả các component con của component đó!**

 Ví dụ, giả sử ta có một component tree `A > B > C > D`, và chúng ta đã xong initial render (đã show ra UI). Sau đó, user click vào một button trong `B` - làm tăng một biến đếm trong component `B`:
 
 - Ta gọi `setState()` trong `B`, làm B bị đánh dấu là cần cập nhật
 - React bắt đầu chạy render từ top của component tree
 - React thấy rằng `A` không bị đánh dấu là cần cập nhật nên bỏ qua nó
 - React thấy rằng `B` bị đánh dấu cần cập nhật và chạy hàm render của `B`. `B` trả về `C`.
 - `C` không được đánh dầu là cần cập nhật. Tuy nhiên, vì parent component của nó là `B` vừa mới re-render, React sẽ render lại child component `C`. `C` trả về `D`.
 - `D` tương tự như trên, dù không được đánh dấu là cần cập nhật nhưng vì `C` re-render nên React cũng thực hiện re-render lại `D`.
 
 Một lần nữa:
 
 **Render một component sẽ, mặc định, khiến cho tất cả các component con bên trong nó re-render luôn!**
 
 Một lưu ý khác:
 
 **Trong quá trình render bình thường, React không quan tâm về "props changed" - nó sẽ re-render tất cả các component con vô điều kiện chỉ vì component cha của chúng bị re-render**
 
 Điều này có nghĩa là gọi `setState()` trong root `<App>` component, sẽ khiến cho tất cả các component trong App bị re-render.
 
 Rất có thể hầu hết các components trong component tree sẽ trả về y chang render output như lần trước đó, và vì thế React không cần cập nhật gì lên real DOM. Nhưng, React vẫn sẽ phải làm công việc là chạy hàm render trên mỗi component, đợi render output và so sánh render output này với render output của lần trước đó - những thứ này sẽ tốn thời gian và năng lực xử lí của CPU.
 
### Component Types và Reconciliation

Như đã được mô tả trong ["Reconciliation" docs page](https://reactjs.org/docs/reconciliation.html#elements-of-different-types), logic render cuả React so sánh các element dựa trên `type` field đầu tiên, dùng phép so sánh `===`. Nếu một element trong một vị trí thay đổi thành một type khác, như từ `<div>` thành `<span>` hay là từ `<ComponentA>` sang `<ComponentB>`, React sẽ tăng tốc quá trình so sánh bằng cách "thôi méo so sánh tiếp nữa" mà giả định rằng cả component đã hay đổi. Kết quả là, React sẽ xóa bỏ tất cả component render output hiện tại, gồm tất cả các DOM nodes (DOM thật), và tạo lại nó từ đầu với một component instance mới.

Điều này có nghĩa rằng bạn không bao giờ được tạo một component type mới trong hàm `render()` (hoặc trong function body của functional component), bởi vì khi bạn tạo một component type mới, nó có một reference mới (vì nó là object mà), điều này sẽ khiến React liên tục xóa và tạo lại cả component sau mỗi lần render.

Nói cách khác, đừng làm thế này:

```javascript
function ParentComponent() {
  // Dòng này sẽ tạo ra một referrence của ChildComponent mỗi lần render!
  function ChildComponent() {}
  
  return <ChildComponent />
}
```

Thay vào đó, luôn define component tách biệt:

```javascript
// Dòng này sẽ chỉ tạo ra 1 component type
function ChildComponent() {}
  
function ParentComponent() {
  return <ChildComponent />
}
```

## Cải thiện hiệu năng Render

Như đã đề cập ở trên, quá trình render của React có thể là dư thừa và gây mất thời gian/tài nguyên ở mỗi lần chạy. Nếu render output của một component không đổi, và không có cập nhật nào cần thiết lên DOM thật, thì quá trình rendering thật sự là lãng phí và thừa thãi.

React component render output khác nhau sẽ dựa trên việc props hiện tại và component state hiện tại có bị thay đổi không. Vì thế, nếu ta biết trước rằng một component props và state sẽ không bị đổi, ta cũng sẽ biết là render ouput sau lần render của component đó sẽ y chang với lần trước, và không có thay đổi nào cần được áp dụng, và ta có thể bỏ qua việc chạy re-render trên component đó.

Khi cố gắng cải thiện hiệu năng phần mềm nói chung, sẽ có 2 cách tiếp cận cơ bản:
   
- Làm hệ thống chạy một task nào đó nhanh hơn (1)
- Làm hệ thống phải chạy ít task hơn (2)

Tối ưu hóa React Rendering chủ yếu là việc cố gắng bỏ qua các lần re-render không cần thiết (2).

### Render Batching và Timing

Mặc định, mỗi lần gọi `setState()` khiến React bắt đầu một quá trình render mới, một cách đồng bộ, và trả về. Tuy nhiên, React cũng ứng dụng một loại tối ưu hóa tự động, được gọi là "render batching". Render batching là React sẽ tự động batch các lần gọi `setState()` liên tiếp nhau và chạy re-render 1 lần thay vì chạy nhiều lần.

React docs có đề cập tới đoạn ["state updates may be asyncronous"](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous), chính là do Render Batching này. Đặc biệt, React tự động batch các state updates xảy ra trong các React event handlers luôn. Vì React event handlers chiếm một lượng lớn code trong các React app thông thường, điều này có nghĩa rằng hầu hết các lần cập nhật state đều được React "batch" lại hết.

React implements render batching cho các event handlers bằng cách wrap chúng lại trong một internal function được gọi là `unstable_batchedUpdates`. React theo dõi tất các các state updates được gọi (gọi `setState()`, ...) khi `unstable_batchedUpdates` đang chạy, và sau đó áp dụng chúng trong một lần render duy nhất.

Về mặt khái niệm, bạn có thể hình dung những gì React đang hoạt động bên trong như đoạn mã giả dưới đây:

```javascript
function internalHandleEvent(e) {
  const userProvidedEventHandler = findEventHandler(e);
  
  let batchedUpdates = [];
  
  unstable_batchedUpdates(() => {
    // mọi state updates được gọi tại đây sẽ được push vào batchedUpdates
    userProvidedEventHandler(e);
  });
  
  renderWithQueuedStateUpdates(batchedUpdates);
}
```

Tuy nhiên, điều này có nghĩa rằng tất cả những state updates mà nằm ngoài immediate call stack của cái event handler đó sẽ không được React batch lại.

Lấy ví dụ sau:

```javascript
const [counter, setCounter] = useState(0);

const onClick = async () => {
  setCounter(0);
  setCounter(1);
  
  const data = await fetchSomeData();
  
  setCounter(2);
  setCounter(3);
}
```

Đoạn code trên sẽ thực hiện 3 lần render. Ở lần render đầu tiên, `setCounter(0)` và `setCounter(1)` được batch và chạy trong cùng 1 lần render, vì cả 2 đều nằm trong immediate call stack của hàm onClick.

Tuy nhiên, với lần gọi `setCounter(2)`, nó nằm sau 1 cái `await`, và nó nằm ngoài immediate call stack của hàm onClick, minh họa bằng code cho dễ hiểu nha:

```javascript
const onClick = async () => {
  setCounter(0);
  setCounter(1);
  
  fetchSomeData(data => {
    setCounter(2);
    setCounter(3);
  });
}
```

Có thể thấy vì hàm `setCounter(2)` và `setCounter(3)` nằm trong phần `.then` của Promise, nên nó sẽ nằm ở một cái event loop callstack khác với hàm onClick (immediate callstack), và vì thế 2 hàm này không được React batch lại, nó sẽ được chạy render một cách đồng bộ, `setCounter(2)` xong rồi tới `setCounter(3)`, là 2 lần re-renders.

> In React's upcoming Concurrent Mode, React will always batch updates, all the time, everywhere.

Một lưu ý nữa là: React sẽ double-render components bên trong thẻ `<StrictMode>` trong development mode, nên bạn không nên dựa vào `console.log()` để đếm số lần re-render của một component. Thay vào đó, hãy dùng React DevTools Profiler để capture tracing, hoặc thêm 1 cái logging vào `useEffect` hook hoặc `componentDidMount/Update` lifecycle - log đó sẽ chỉ được in ra khi React thực sự hoàn thành render và commit changes vào DOM thật.

### Các kĩ thuật tối ưu hóa cho Component Render

React cung cấp cho chúng ta 3 APIs để cho phép bỏ qua quá trình re-render trên một component:

- `React.Component.shouldComponentUpdate`: là một optional class component lifecycle method sẽ được gọi trước khi render process diễn ra. Nếu method này trả về `false`, React sẽ bỏ qua việc re-render component. Một cách sử dụng phổ biến của method này là kiểm tra nếu component props và state thay đổi hay chưa.
- `React.PureComponent`: đây là một Base Class thay thế cho `React.Component`, implement sẵn hàm `shouldComponentUpdate` bằng cách so sánh props và state mới với cũ.
- `React.memo()` là một built-in "higher order component". Nó nhận vào tham số là một component, và trả về một wrapper component. Default behavior của wrapper component này là kiểm tra props có bị đổi không, và nếu không thì ngăn chặn re-render. Cả functional component và class component đều có thể được wrap bởi `React.memo()`.

Tất cả các cách tiếp cận trên dùng một kĩ thuật so sánh được gọi là "shallow equality" (so sánh nông). Có nghĩa là nó sẽ kiểm tra tất cả các field riêng lẻ trong 2 objects xem có cùng value không. Nói cách khác, `obj1.a === obj2.a && object1.b === object2.b && ...`.

Ngoài ra còn một kĩ thuật tối ưu ít được biết đến hơn của React: nếu một React component trả về render output là element reference giống với lần trước đó, React sẽ bỏ qua việc re-render.

Cho tất cả các kĩ thuật này, bỏ qua re-render một component đồng nghĩa với việc React sẽ cũng bỏ qua render trên cả subtree element của component đó ("render children recursively" behavior).

### Xài References cho new Props ảnh hưởng thế nào tới việc tối ưu hóa Render

Chúng ta đều biết rằng mặc định, React re-render tất cả nested component kể cả khi props của chúng không đổi. Cũng có nghĩa rằng truyền props là new references vào một component con là không sao cả, bởi vì nó cũng sẽ re-render cho dù bạn có truyền props giống nhau hay không. Ví dụ như đoạn code dưới đây là ổn:

```javascript
function ParentComponent() {
  const onClick = () => {
    console.log("Button clicked")
  }

  const data = {a: 1, b: 2}

  return <NormalChildComponent onClick={onClick} data={data} />
}
```

Mỗi lần `ParentComponent` re-render, nó sẽ tạo một `onClick` function reference và một `data` object reference mới, sau đó truyền vào dưới dạng props cho `NormalChildComponent`. (Lưu ý rằng việc define hàm `onClick` bằng từ khóa `function` hay bằng `arrow function` không khác nhau trong trường hợp này).

Điều đó cũng có nghĩa rằng không có ích gì khi cố gắng tối ưu hóa cho các "component cơ bản", như một `<div>` hay một `<button>`, bằng cách wrap chúng vào `React.memo()`. Không có một component con nào bên dưới các component cơ bản đó, nên quá trình re-render sẽ dừng lại tại đó bất kể trong trường hợp nào.

(WIP)

### Tối ưu hóa Props Reference

Nếu xài class component bạn không cần lo lắng khi tạo lại reference mới cho các callbacks, bởi vì chúng có thể có instance methods mà luôn có chung một reference. Tuy nhiên, những component instance này sẽ cần tạo unique callbacks cho những child list item tách biệt, hoặc capture một giá trị trong một anonymous function và truyền vào component con. Tất cả những điều đó đều dẫn tới kết quả là phải tạo reference mới mỗi khi re-render, và như vậy sẽ tạo ra thêm object và child props mới khi re-render -> đồng nghĩa với việc sẽ tốn thêm một ít bộ nhớ sau mỗi lần re-render, GC phải hoạt động thường xuyên hơn. **Với class-components, React không cung cấp một built-in feature nào để tối ưu hóa những trường hợp này.**

Đối với functional component, React cung cấp 2 hooks để giúp bạn tái sử dụng reference: `useMemo` khi tạo object mới hay thực hiện các phép tính phức tạp, và `useCallback` để tái sử dụng các callback functions.

### Memoize tất cả mọi thứ?

Như đã đề cập ở trên, bạn không cần phải dùng `useMemo` và `useCallback` ở mỗi hàm hay object mà bạn truyền xuống dưới dạng props - chỉ cần thiết khi nó tạo ra sự khác biệt trong behavior của component con. Thường chỉ sử dụng `useMemo` và `useCallback` đối với những object phức tạp và có vẻ sẽ tốn nhiều bộ nhớ, vì sử dụng sai dễ dẫn tới việc bugs component không được re-render sau khi update props/state (khi mutate object, sẽ đề cập ở dưới).

Một câu hỏi khác hay được hỏi nhiều là "Tại sao mặc định React không wrap mọi thứ bằng `React.memo`?"

Dan Abramov đã [nhiều lần chỉ ra rằng dù là memoization thì bạn vẫn phải tốn chi phí cho việc so sánh props: độ phức tạp O(prop count)](https://twitter.com/dan_abramov/status/1095661142477811717), và trong một số trường hợp thì memoization không bao giờ có thể ngăn chặn re-render vì component luôn nhận vào props mới. Lấy ví dụ, [xem Twitter thread này của Dan](https://twitter.com/dan_abramov/status/1083897065263034368):

> Why doesn’t React put memo() around every component by default? Isn’t it faster? Should we make a benchmark to check?

> Ask yourself:
  
> Why don’t you put Lodash memoize() around every function? Wouldn’t that make all functions faster? Do we need a benchmark for this? Why not?

Ngoài ra, việc mặc định áp dụng `React.memo()` cho tất cả component sẽ dẫn tới kết quả là tạo ra bug trong những trường hợp developer cố ý/vô tình mutate object (`obj.a = 'changed'`) thay vì cập nhật nó theo cách immutable (`obj = {...obj, a: 'changed' }`).

### Immutability và Rerendering

**State update trong React nên được thực hiện một cách immutably.** Đây là 2 lí do chính:

- Tùy thuộc vào những gì bạn thay đổi và ở đâu, nó có thể dẫn đến các component không được re-render như mong đợi
- Nó dễ tạo ra nhầm lẫn về `ở đâu` và `vì sao` mà data được update.

Để dễ hiểu hơn, hãy xem qua một số ví dụ sau.

Như chúng ta đã biết, `React.memo` / `PureComponent` / `shouldComponentUpdate` tất cả đều phụ thuộc vào shallow equality (so sánh nông) props hiện tại và props trước đó. Chúng ta mong đợi rằng nếu props thay đổi thì `props.someValue !== prevProps.someValue`.

Nếu bạn mutate, thì reference của `someValue` hiện tại sẽ giống với reference `someValue` cũ, và `shouldComponentUpdate` sẽ trả về false, mọi thứ sẽ không được re-render và dẫn tới bugs.

Một vấn đề khác là với `useState` và `useReducer` hooks. Mỗi lần bạn gọi `setCounter()` hoặc `dispatch()`, React sẽ chuẩn bị quá trình re-render. Tuy nhiên, React đòi hỏi rằng tất cả các hooks state update phải được truyền vào giá trị là một reference mới, nếu bạn truyền vào một reference là một object/array cũ chẳng hạn, component cũng sẽ không được re-render, ví dụ như này:

```javascript
const [todos, setTodos] = useState(someTodosArray);

const onClick = () => {
  todos[3].completed = true;
  setTodos(todos);
}
```

Thì component sẽ không thể re-render, cần phải truyền vào `setTodos` là một reference mới.

```javascript
const onClick = () => {
  const newTodos = todos.slice();
  newTodos[3].completed = true;
  setTodos(newTodos);
}
```

Lưu ý rằng có một sự khác biệt rõ ràng về behavior của class component `this.setState()` và functional component `useState` / `useReducer` về mutations và re-rendering: `this.setState` không quan tâm về việc bạn mutate hay truyền vào 1 reference mới, nó luôn thực hiện re-render, ví dụ đoạn code dưới đây vẫn sẽ re-render:

```javascript
const {todos} = this.state;
todos[3].completed = true;
this.setState({todos});
```

**Nói tóm lại: React, và React ecosystem, giả định rằng tất cả các update đều là immutable. Bất cứ khi nào bạn mutate state, app của bạn sẽ có nguy cơ bị lỗi. Đừng bao giờ mutate!**

### Đo lường hiệu năng React Component Rendering

Sử dụng [React DevTools Profiler](https://reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html) để trace việc components được render như thế nào mỗi ở mỗi commit. Tìm ra các components re-render lại quá nhiều lần và dùng DevTools để tìm hiểu vì sao nó re-render và fix component đó (bằng cách wrap lại trong `React.memo` chẳng hạn, hoặc memoize props trong parent component).

## Context và Rendering Behavior

React's Context API là một cơ chế để tạo ra một state value (có thể là object, array hoặc một primitive value) có thể được truy cập bởi một subtree components, bất kì component nào nằm trong wrapper của Context `<MyContext.Provider>` có thể đọc value từ context instance đó, mà không cần phải truyền props lần lượt từ trên xuống dưới.

**Context không phải là một "state management" tool**. Do vậy bạn phải tự quản lí tất cả values được pass vào context. Điều này thường được thực hiện bằng cách giữ data trong React component state, và constructing context values dựa trên data đó.

### Context cơ bản

Context provider nhận vào một `value` prop, như là `<MyContext.Provider value={42}>`. Các component con có thể consume context đó bằng cách render context consumer và lấy value trong context bằng một render prop, ví dụ như sau:

```jsx
<MyContext.Provider>
    {(value) => <div>{value}</div>}
</MyContext.Provider>
```

Hoặc với functional component, có thể xài `useContext` hook:

```javascript
const value = useContext(MyContext);
```

### Cập nhật Context Values

React kiểm tra nếu value trong context provider bị thay đổi khi các component bọc bên ngoài context-provider re-render. Nếu value của context-provider là một reference mới, thì React sẽ biết được là value đã bị thay đổi, và những components consume context đó cần được cập nhật.

Lưu ý rằng truyền new object vào context provider sẽ khiến nó update:

```javascript
function GrandchildComponent() {
  const value = useContext(MyContext);
  return <div>{value.a}</div>
}

function ChildComponent() {
  return <GrandchildComponent />
}

function ParentComponent() {
  const [a, setA] = useState(0);
  const [b, setB] = useState("text");

  const contextValue = {a, b};

  return (
    <MyContext.Provider value={contextValue}>
      <ChildComponent />
    </MyContext.Provider>
  )
}
```

Trong ví dụ trên, mỗi khi `ParentComponent` re-render, React sẽ hiểu rằng `MyContext.Provider` đã được thay đổi giá trị mới (mặc dù có thể thật sự là không), và tìm các components đang consume `MyContext` để đánh dấu là cần cập nhật. **Khi một context provider có giá trị mới (giá trị reference mới), tất cả nested component đang consume context đó sẽ bị forced re-render.**

Hiện tại, không có cách nào cho phép một component đang consume một context bỏ qua việc re-render do context value cập nhật value mới.

### State Update, Context và Re-Renders

Tóm tắt lại một số ý từ đầu tới giờ:

- Gọi `setState()` sẽ khiến component bị re-render
- React sẽ mặc định render lặp đệ quy xuống tất cả các nested components
- Context provider được cung cấp value từ component render nó
- Value đó thường đến từ component cha của component render context provider

Điều này có nghĩa rằng mặc định thì, mọi update đến parent component mà render context provider sẽ khiến cho tất cả component con của nó re-render, bất kể những components đó có đọc value từ context hay không.

Nếu nhìn lại cái ví dụ `Parent/Child/Grandchild` ở trên, ta có thể thấy `GrandchildComponent` sẽ được re-render, nhưng không phải là do context-update - mà nó re-render là do `ChildComponent` re-render. Trong ví dụ ở trên không có một optimization nào được thể hiện để loại bỏ "unnecessary" re-renders, nên React sẽ mặc định re-render `ChildComponent` và `GrandchildComponent` mỗi khi `ParentComponent` re-render. Nếu `ParentComponent` thay đổi giá trị trong `MyContext.Provider`, `GrandchildComponent` sẽ đọc được giá trị mới khi nó re-render và dùng nó, nhưng không phải việc update context khiến cho nó re-render.

### Context Updates và Tối ưu hóa Re-render

Hãy thử sửa ví dụ ở trên để thực hiện một vài tối ưu hóa cho việc re-render, chúng ta cũng sẽ thêm một `GreatGrandchildComponent` ở dưới cùng:

```javascript
function GreatGrandchildComponent() {
  return <div>Hi</div>
}

function GrandchildComponent() {
    const value = useContext(MyContext);
    return (
      <div>
        {value.a}
        <GreatGrandchildComponent />
      </div>
}

function ChildComponent() {
    return <GrandchildComponent />
}

const MemoizedChildComponent = React.memo(ChildComponent);

function ParentComponent() {
    const [a, setA] = useState(0);
    const [b, setB] = useState("text");
    
    const contextValue = {a, b};
    
    return (
      <MyContext.Provider value={contextValue}>
        <MemoizedChildComponent />
      </MyContext.Provider>
    )
}
```

OK bây giờ, nếu ta gọi `setA(42)`:

- `ParentComponent` sẽ re-render
- Một `contextValue` reference mới sẽ được tạo
- React thấy rằng `MyContext.Provider` được cập nhật value mới, nên các consumer của `MyContext` sẽ cần được cập nhật. (1)
- React sẽ thử re-render `MemoizedChildComponent`, nhưng vì nó được wrap trong `React.memo()`, vì không có props nào được truyền vào, nên sẽ không có props nào đổi. React sẽ bỏ qua việc render `ChildComponent`.
- Tuy nhiên, vì value của `MyContext` đã bị thay đổi (do 1), nên sẽ có một số component consume cần được biết.
- React tiếp tục lặp xuống dưới và thấy rằng `GrandchildComponent` đang đọc value trong `MyContext`, nên `GrandchildComponent` cần được re-render. `GrandchildComponent` sau đó được re-render, vì context value change.
- Vì `GrandchildComponent` render, component con của nó là `GreatGrandchildComponent` cũng sẽ bị re-render theo.

Nói cách khác, như trong [twitt của Sophie Alpert](https://twitter.com/sophiebits/status/1228942768543686656):

> That React Component Right Under Your Context Provider Should Probably Use React.memo

## Redux và Rendering Behavior

"Xài Redux hay Context?" có vẻ là câu hỏi được nhắc tới nhiều nhất trong các cuộc tranh luận trong cộng đồng React dạo gần đây, sau khi React Hooks trở nên phổ biến. Thực tế là Redux và Context là 2 công cụ khác nhau để làm những việc khác nhau, chi tiết sẽ được nêu ra bên dưới.

Một kết luận hay được nêu ra trong các cuộc tranh luận giữa Context và Redux là: "Redux chỉ re-render những components thật sự cần re-render, nên hiệu năng của nó tốt hơn Context" ("context makes everything render, Redux doesn't, use Redux").

Điều đó có phần đúng, nhưng câu trả lời thực ra còn mang nhiều sắc thái hơn thế.

### Redux Subscriptions

Có nhiều người nói rằng "Redux thực ra cũng dùng Context bên dưới thôi", thực ra cũng phần nào đúng, [nhưng **Redux dùng Context để truyền Redux store instance, chứ không phải là state value**](https://blog.isquaredsoftware.com/2020/01/blogged-answers-react-redux-and-context-behavior/). Điều đó có nghĩa là Redux luôn truyền một context value không thay đổi vào `<ReactReduxContext.Provider>` trong suốt quá trình App chạy.

Redux store chạy tất cả subscriber notification callbacks mỗi khi có 1 action được dispatch. React Component cần sử dụng Redux luôn subscribe vào Redux store, đọc giá trị mới nhận từ subscriber callbacks, so sánh giá trị đó, và force re-render nếu giá trị mới khác với giá trị hiện tại. Hàm subscription callback được xử lí bên ngoài React app (không được managed bởi React), và React app chỉ được thông báo khi có một React Component subscribe tới một data trong Redux, mà data đó vừa bị thay đổi (dựa trên giá trị trả về của `mapState` hoặc `useSelector`).

Behavior này của React dẫn đến các đặc điểm về hiệu năng rất khác với Context. Đúng vậy, có vẻ là sẽ có ít components phải re-render hơn xuyên suốt quá trình chạy, nhưng Redux sẽ phải luôn chạy hàm `mapState/useSelector` trên toàn bộ component tree (nghĩa là sẽ chạy hàm đó trên mỗi component subscribe vào Redux store) mỗi khi store state được cập nhật. **Hầu hết các trường hợp, chi phí chạy những hàm selector này ít hơn chi phí cho React thực hiện các phase re-render, nên nó thường mang lại nhiều lợi ích hơn về mặt hiệu năng cho app**, tuy nhiên nếu những hàm selector có bao gồm những hàm tính toán phức tạp, các transformations tốn kém hoặc vô tình luôn luôn trả về giá trị mới thì mọi thứ có thể bị làm chậm đi.

### Khác biệt giữa `connect` và `useSelector`

`connect` là 1 higher-order component, nó trả về một wrapper component thực hiện tất cả công việc từ subscribe tới store, chạy `mapState` và `mapDispatch`, và truyền combined props từ store xuống component của bạn.

Wrapper component `connect` luôn hoạt động tương tự `PureComponent/React.memo()`, nhưng với một điểm khác: `connect` sẽ chỉ làm component của bạn re-render nếu combined props được truyền xuống component bị thay đổi. Thông thường, combined props cuối cùng được truyền xuống thường là một sự phối hợp giữa các `{...ownProps, ...stateProps, ...dispatchProps}`, nên mỗi props reference mới từ parent component truyền xuống sẽ khiến component của bạn re-render, tương tự như `PureComponent/React.memo()`. Bên cạnh parent props, [mỗi reference mới được trả về từ `Mapstate` cũng sẽ khiến component re-render](https://react-redux.js.org/using-react-redux/connect-mapstate#mapstatetoprops-and-performance) (vì thế bạn có thể customize cách mà `ownProps/stateProps/dispatchProps` được merged, để thay đổi re-render behavior này).

Mặt khác, `useSelector` là 1 hook được gọi **bên trong** functional component. Vì thế, `useSelector` không có cách nào ngăn chặn component của bạn re-render khi parent component của nó re-render!

Đây là [một khác biệt then chố về hiệu năng giữa `connect` và `useSelector`](https://react-redux.js.org/api/hooks#performance). Với `connect`, mọi connected component sẽ hoạt động như `PureComponent`.

Vì vậy, nếu bạn chỉ sử dụng các functional components và `useSelector`, thì có khả năng là phần lớn các components trong App sẽ re-render nhiều hơn theo các thay đổi từ Redux store hơn là khi bạn xài `connect`, vì sẽ không có gì chặn việc component con re-render sau khi component cha của nó re-render.

Nếu điều đó trở thành một vấn đề về hiệu năng (quá nhiều re-renders), thì bạn nên wrap các functional component bằng `React.memo()` để chặn bớt việc component con re-render theo component cha.
