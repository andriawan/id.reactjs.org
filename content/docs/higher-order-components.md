---
id: higher-order-components
title: Higher-Order Components
permalink: docs/higher-order-components.html
---

_Higher-order component_ (HOC) merupakan teknik lanjutan dalam React untuk menggunakan kembali logika komponen. HOCs sendiri bukan merupakan bagian dari API React. Hal tersebut merupakan pola yang muncul dari sifat komposisi React.

Konkritnya, **_higher-order component_ merupakan fungsi yang mengambil sebuah komponen dan mengembalikan sebuah komponen baru**

```js
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```

Sebaliknya saat sebuah komponen mengubah _props_ menjadi UI, _higher-order component_ mengubah sebuah komponen menjadi komponen yang lainnya.

HOCs umum di pakai pustaka pihak ketiga React, seperti Redux's [`connect`](https://github.com/reduxjs/react-redux/blob/master/docs/api/connect.md#connect) dan Relay's [`createFragmentContainer`](http://facebook.github.io/relay/docs/en/fragment-container.html).

Pada dokumen ini, kita akan mendiskusikan mengapa _higher-order components_ bermanfaat dan bagaimana menulis _higher-order components_ anda sendiri.

## Penggunaan HOCs untuk Cross-Cutting Concerns {#use-hocs-for-cross-cutting-concerns}

> **Catatan**
>
> Kita sebelumnya merekomendasikan _mixins_ sebagai cara menangani _cross-cutting concerns_. Kita telah menyadari bahwa _mixins_ menimbulkan lebih banyak masalah daripada keuntungan. [Baca detail](/blog/2016/07/13/mixins-considered-harmful.html) tentang mengapa kita beralih dari _mixins_ dan bagaimana Anda dapat mentransisikan komponen yang ada.

Komponen merupakan unit utama dari penggunaan ulang kode di React. Namun, Anda akan menemukan bahwa beberapa pola tidak cocok untuk komponen tradisional.

Contohnya, Anda memiliki komponen `CommentList` yang berlangganan ke sumber data eksternal untuk me-_render_ daftar komentar:

```js
class CommentList extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      // "DataSource" is some global data source
      comments: DataSource.getComments()
    };
  }

  componentDidMount() {
    // Subscribe to changes
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    // Clean up listener
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    // Update component state whenever the data source changes
    this.setState({
      comments: DataSource.getComments()
    });
  }

  render() {
    return (
      <div>
        {this.state.comments.map((comment) => (
          <Comment comment={comment} key={comment.id} />
        ))}
      </div>
    );
  }
}
```

Kemudian, Anda menulis sebuah komponen untuk berlangganan ke posting blog yang mengikuti pola yang sama:

```js
class BlogPost extends React.Component {
  constructor(props) {
    super(props);
    this.handleChange = this.handleChange.bind(this);
    this.state = {
      blogPost: DataSource.getBlogPost(props.id)
    };
  }

  componentDidMount() {
    DataSource.addChangeListener(this.handleChange);
  }

  componentWillUnmount() {
    DataSource.removeChangeListener(this.handleChange);
  }

  handleChange() {
    this.setState({
      blogPost: DataSource.getBlogPost(this.props.id)
    });
  }

  render() {
    return <TextBlock text={this.state.blogPost} />;
  }
}
```

`CommentList` dan `BlogPost` tidaklah sama — Keduanya memanggil metode yang berbeda `DataSource`, dan keduanya me-_render_ keluaran yang berbeda. Namun, implementasinya kebanyakan sama:

- Saat dilakukan _mount_, tambah _change listener_ ke `DataSource`.
- Di dalam _listener_, panggil `setState` pada saat  sumber data berubah.
- Saat dilakukan _unmount_, hapus _change listener_.

Anda dapat bayangkan bahwa dalam aplikasi berskala besar, pola yang sama pada proses berlangganan `DataSource` dan pemanggilan `setState` akan terjadi berulang kali. Kita ingin sebuah abstraksi yang mengizinkan kita mendefinisikan logika ini pada satu tempat dan membaginya antar komponen. Dalam kondisi ini, _higher-order components_ unggul.

Kita dapat menulis sebuah fungsi yang membuat komponen, seperti `CommentList` dan `BlogPost` yang berlangganan ke `DataSource`. Fungsi tersebut akan menerima salah satu argumennya ialah komponen turunan yang menerima data langganan sebagai _prop_s. Mari kita panggil fungsi `withSubscription`:

```js
const CommentListWithSubscription = withSubscription(
  CommentList,
  (DataSource) => DataSource.getComments()
);

const BlogPostWithSubscription = withSubscription(
  BlogPost,
  (DataSource, props) => DataSource.getBlogPost(props.id)
);
```

Parameter pertama ialah komponen yang terbungkus. Parameter kedua mengambil data yang kita inginkan, contohnya  ialah `DataSource` dan _props_ saat ini.

Saat `CommentListWithSubscription` dan `BlogPostWithSubscription` di-_render_, `CommentList` dan `BlogPost` akan dioper sebuah `data` _prop_ dengan data paling baru yang diperoleh dari `DataSource`:

```js
// This function takes a component...
function withSubscription(WrappedComponent, selectData) {
  // ...and returns another component...
  return class extends React.Component {
    constructor(props) {
      super(props);
      this.handleChange = this.handleChange.bind(this);
      this.state = {
        data: selectData(DataSource, props)
      };
    }

    componentDidMount() {
      // ... that takes care of the subscription...
      DataSource.addChangeListener(this.handleChange);
    }

    componentWillUnmount() {
      DataSource.removeChangeListener(this.handleChange);
    }

    handleChange() {
      this.setState({
        data: selectData(DataSource, this.props)
      });
    }

    render() {
      // ... and renders the wrapped component with the fresh data!
      // Notice that we pass through any additional props
      return <WrappedComponent data={this.state.data} {...this.props} />;
    }
  };
}
```

Catat bahwa sebuah HOC tidak mengubah komponen _input_, tidak pula menggunakan _inheritance_ untuk menyalin perilakunya. Sebaliknya, sebuah HOC *menyusun* komponen asli dengan cara *membungkusnya* ke dalam sebuah _container_. Sebuah HOC merupakan fungsi murni bebas dari _side-effects_.

Dan jadilah! komponen yang terbungkus menerima semua _props_ dari _container_ sejalan dengan _prop_ baru, `data`, yang mana digunakan untuk me-_render_ keluaranya. HOC tidak memperhatikan bagaimana data digunakan dan komponen yang terbungkus tidak memperhatikan darimana data berasal.

Karena `withSubscription` merupakan fungsi normal, Anda dapat menambahkan sebanyak atau pun sesedikit mungkin argumen yang anda inginkan. Contohnya, Anda ingin membuat nama dari `data` _props_ dapat diatur untuk nantinya dapat mengisolasi HOC dari komponen yang terbungkus. Atau Anda dapat menerima sebuah argumen yang mengatur `shouldComponentUpdate`, atau satu yang mengatur sumber data. Hal ini memungkinkan karena HOC memiliki kontrol penuh terhadap bagaimana komponen didefinisikan.

Seperti komponen, kontrak antara `withSubscription` dan komponen yang terbungkus seluruhnya merupakan _props-based_. Hal ini memudahkan bertukar dari satu HOC ke yang lainnya, selama menyediakan _props_ yang sama ke komponen yang terbungkus. Hal ini berguna contohnya jika Anda mengubah pustaka _data-fetching_.

## Jangan memutasi komponen asli. Gunakan _Composition_. {#dont-mutate-the-original-component-use-composition}

Tahan godaan untuk memodifikasi prototipe komponen (jika tidak, lakukan mutasi) di dalam  HOC.

```js
function logProps(InputComponent) {
  InputComponent.prototype.componentWillReceiveProps = function(nextProps) {
    console.log('Current props: ', this.props);
    console.log('Next props: ', nextProps);
  };
  // The fact that we're returning the original input is a hint that it has
  // been mutated.
  return InputComponent;
}

// EnhancedComponent will log whenever props are received
const EnhancedComponent = logProps(InputComponent);
```

Ada sedikit masalah dengan kode ini. Salah satunya ialah komponen masukan tidak dapat digunakan kembali secara terpisah dari komponen yang ditingkatkan. lebih petingnya lagi, jika Anda menerapkan HOC yang lain ke `EnhancedComponent` yang *juga* memutasi `componentWillReceiveProps`, fungsionalitas HOC pertama akan ditimpa! HOC ini juga tidak akan bekerja dengan fungsional komponen, yang mana tidak memiliki sikuls metode.

Mengubah HOC merupakan kebocoran abstraksi - pengguna harus mengerti bagaimana mereka diimplementasikan untuk menghindari konflik dengan HOC lainnya.

Daripada mutasi, HOC seharusnya menggunakan _composition_, dengan membungkus komponen masukan ke dalam _container_ komponen:

```js
function logProps(WrappedComponent) {
  return class extends React.Component {
    componentWillReceiveProps(nextProps) {
      console.log('Current props: ', this.props);
      console.log('Next props: ', nextProps);
    }
    render() {
      // Wraps the input component in a container, without mutating it. Good!
      return <WrappedComponent {...this.props} />;
    }
  }
}
```

HOC memiliki fungsionalitas yang sama dengan versi mutasi sembari menghindari potensi bentrok. Hal itu berfungsi sama baiknya dengan kelas dan fungsional komponen. Dan karena merupakan fungsi murni, hal tersebut dapat disusun dengan komponen HOC lainnya, atau bahkan dengan komponen itu sendiri.

Anda mungkin memperhatikan kemiripan antara HOC dan pola yang disebut *komponen container*. *Komponen container* merupakan bagian dari strategi pemisahan *responsibility* antara kepentingan *high-level* dan *low-level*. Container menangani hal seperti langganan dan *state*, dan mengoper ke komponen yang menangani hal seperti *rendering UI*. HOC menggunakan *container* sebagai bagian dari implementasinya. Anda dapat berfikir bahwa HOC merupakan *komponen container* terdefinisi yang berparameter.

## Persetujuan: Oper _Props_ yang terkait melalui komponen yang dibungkus {#convention-pass-unrelated-props-through-to-the-wrapped-component}

HOC menambahkan fitur ke komponen. Mereka tidak seharusnya secara drastis mengubah kontraknya. Diharapkan bahwa komponen yang dikembalikan dari HOC memiliki antarmuka yang mirip dengan komponen yang dibungkus.

HOC seharusnya mengoper melalui _props_ yang tidak terkait ke perhatian khususnya. Sebagian besar HOC berisi metode _render_ yang terlihat seperti ini:

```js
render() {
  // Filter out extra props that are specific to this HOC and shouldn't be
  // passed through
  const { extraProp, ...passThroughProps } = this.props;

  // Inject props into the wrapped component. These are usually state values or
  // instance methods.
  const injectedProp = someStateOrInstanceMethod;

  // Pass props to wrapped component
  return (
    <WrappedComponent
      injectedProp={injectedProp}
      {...passThroughProps}
    />
  );
}
```

This convention helps ensure that HOCs are as flexible and reusable as possible.

## Convention: Maximizing Composability {#convention-maximizing-composability}

Not all HOCs look the same. Sometimes they accept only a single argument, the wrapped component:

```js
const NavbarWithRouter = withRouter(Navbar);
```

Usually, HOCs accept additional arguments. In this example from Relay, a config object is used to specify a component's data dependencies:

```js
const CommentWithRelay = Relay.createContainer(Comment, config);
```

The most common signature for HOCs looks like this:

```js
// React Redux's `connect`
const ConnectedComment = connect(commentSelector, commentActions)(CommentList);
```

*What?!* If you break it apart, it's easier to see what's going on.

```js
// connect is a function that returns another function
const enhance = connect(commentListSelector, commentListActions);
// The returned function is a HOC, which returns a component that is connected
// to the Redux store
const ConnectedComment = enhance(CommentList);
```
In other words, `connect` is a higher-order function that returns a higher-order component!

This form may seem confusing or unnecessary, but it has a useful property. Single-argument HOCs like the one returned by the `connect` function have the signature `Component => Component`. Functions whose output type is the same as its input type are really easy to compose together.

```js
// Instead of doing this...
const EnhancedComponent = withRouter(connect(commentSelector)(WrappedComponent))

// ... you can use a function composition utility
// compose(f, g, h) is the same as (...args) => f(g(h(...args)))
const enhance = compose(
  // These are both single-argument HOCs
  withRouter,
  connect(commentSelector)
)
const EnhancedComponent = enhance(WrappedComponent)
```

(This same property also allows `connect` and other enhancer-style HOCs to be used as decorators, an experimental JavaScript proposal.)

The `compose` utility function is provided by many third-party libraries including lodash (as [`lodash.flowRight`](https://lodash.com/docs/#flowRight)), [Redux](https://redux.js.org/api/compose), and [Ramda](https://ramdajs.com/docs/#compose).

## Convention: Wrap the Display Name for Easy Debugging {#convention-wrap-the-display-name-for-easy-debugging}

The container components created by HOCs show up in the [React Developer Tools](https://github.com/facebook/react-devtools) like any other component. To ease debugging, choose a display name that communicates that it's the result of a HOC.

The most common technique is to wrap the display name of the wrapped component. So if your higher-order component is named `withSubscription`, and the wrapped component's display name is `CommentList`, use the display name `WithSubscription(CommentList)`:

```js
function withSubscription(WrappedComponent) {
  class WithSubscription extends React.Component {/* ... */}
  WithSubscription.displayName = `WithSubscription(${getDisplayName(WrappedComponent)})`;
  return WithSubscription;
}

function getDisplayName(WrappedComponent) {
  return WrappedComponent.displayName || WrappedComponent.name || 'Component';
}
```


## Caveats {#caveats}

Higher-order components come with a few caveats that aren't immediately obvious if you're new to React.

### Don't Use HOCs Inside the render Method {#dont-use-hocs-inside-the-render-method}

React's diffing algorithm (called reconciliation) uses component identity to determine whether it should update the existing subtree or throw it away and mount a new one. If the component returned from `render` is identical (`===`) to the component from the previous render, React recursively updates the subtree by diffing it with the new one. If they're not equal, the previous subtree is unmounted completely.

Normally, you shouldn't need to think about this. But it matters for HOCs because it means you can't apply a HOC to a component within the render method of a component:

```js
render() {
  // A new version of EnhancedComponent is created on every render
  // EnhancedComponent1 !== EnhancedComponent2
  const EnhancedComponent = enhance(MyComponent);
  // That causes the entire subtree to unmount/remount each time!
  return <EnhancedComponent />;
}
```

The problem here isn't just about performance — remounting a component causes the state of that component and all of its children to be lost.

Instead, apply HOCs outside the component definition so that the resulting component is created only once. Then, its identity will be consistent across renders. This is usually what you want, anyway.

In those rare cases where you need to apply a HOC dynamically, you can also do it inside a component's lifecycle methods or its constructor.

### Static Methods Must Be Copied Over {#static-methods-must-be-copied-over}

Sometimes it's useful to define a static method on a React component. For example, Relay containers expose a static method `getFragment` to facilitate the composition of GraphQL fragments.

When you apply a HOC to a component, though, the original component is wrapped with a container component. That means the new component does not have any of the static methods of the original component.

```js
// Define a static method
WrappedComponent.staticMethod = function() {/*...*/}
// Now apply a HOC
const EnhancedComponent = enhance(WrappedComponent);

// The enhanced component has no static method
typeof EnhancedComponent.staticMethod === 'undefined' // true
```

To solve this, you could copy the methods onto the container before returning it:

```js
function enhance(WrappedComponent) {
  class Enhance extends React.Component {/*...*/}
  // Must know exactly which method(s) to copy :(
  Enhance.staticMethod = WrappedComponent.staticMethod;
  return Enhance;
}
```

However, this requires you to know exactly which methods need to be copied. You can use [hoist-non-react-statics](https://github.com/mridgway/hoist-non-react-statics) to automatically copy all non-React static methods:

```js
import hoistNonReactStatic from 'hoist-non-react-statics';
function enhance(WrappedComponent) {
  class Enhance extends React.Component {/*...*/}
  hoistNonReactStatic(Enhance, WrappedComponent);
  return Enhance;
}
```

Another possible solution is to export the static method separately from the component itself.

```js
// Instead of...
MyComponent.someFunction = someFunction;
export default MyComponent;

// ...export the method separately...
export { someFunction };

// ...and in the consuming module, import both
import MyComponent, { someFunction } from './MyComponent.js';
```

### Refs Aren't Passed Through {#refs-arent-passed-through}

While the convention for higher-order components is to pass through all props to the wrapped component, this does not work for refs. That's because `ref` is not really a prop — like `key`, it's handled specially by React. If you add a ref to an element whose component is the result of a HOC, the ref refers to an instance of the outermost container component, not the wrapped component.

The solution for this problem is to use the `React.forwardRef` API (introduced with React 16.3). [Learn more about it in the forwarding refs section](/docs/forwarding-refs.html).
