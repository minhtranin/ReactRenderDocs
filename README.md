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

Tất cả các cách tiếp c�
