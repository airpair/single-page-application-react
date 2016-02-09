`The React framework has very clearly been developed to take advantage of recent advantages in the Javascript language ecosystem (ES2015). This gives it a clear edge over most of the competition. On top of that it provides a very elegant way to write complex components in a maintainable fashion. This article will attempt to provide an explanation and an example to get you going on your react journey.`

# ReactJS Reusable Components using Redux and Immutable

## 1. The unique challenges of client side development
There are well established practices for reusing and sharing serverside code. Very frequently however, client side code is written in a slapdash, ad-hoc manner, with no real reuse built into the process. It is frequently either a rigid, static unit fulfilling a concrete problem, or an illegible mess of code reflection and metaprogramming. 
Additionally, client side development is inherently stateful, relying on global state mutations and side effects via browser rendering, and server calls, making debugging difficult, and making referential transparency and idempotence almost impossible to achieve.
Finally, while the Javascript Specification is evolving to improve the language, client side development frequently cannot take advantage of this progress due to browser support requirements which limit us to using the outdated ES5 standard we "know and love" `</sarcasm>`
This article will attempt to outline an approach to development that mitigates at least some of these issues, using the React framework and a host of utility and helper libraries.
Please note, that the goal of this article is not styling, and as such CSS will be kept to a minimum.

### 1.1 Target Audience.
This article is aimed squarely at front end engineers. Solid experience working with a client side framework like Angular or Ember would be incredibly advantageous to help frame this discussion.  Some experience with React and ES2015 would be helpful but not strictly required. Having used npm and the node ecosystem in general would be beneficial.

##2 The stack, in brief.
This article will cover the development of client side components using React, Redux, Immutable.JS,  and ES2015. We will use Babel for transpilation, and npm as a script runner and package tool. All code used in this article can be found at the <sup><a href="https://github.com/apolishch/react-article">github repo associated with this article</a></sup>

##3 Flux architecture and design principles
Redux is one of many possible implementations of the flux architecture. It is an architecture invented at Facebook and used with React instead of the more traditional MVC approach. The core design principle of a good Flux application is *unidirectional data flow*. What this means is that data should flow one way, from the store to the component. The component should issue actions, which should make changes to the system state by way of (for example) API calls, and trigger store changes. A central dispatcher should route these actions to the relevant stores. You should never manipulate the store directly from a component, nor issue actions from a store. In the interests of context, a reasonably good mapping to more traditional MVC would map as follows:

Store <-> Model
Actions/Dispatcher <-> Services/Controller
Component <-> View

###3.1 Redux
Redux differs from this design philosophy in two, fundamental ways. First, there is only one store, negating the need for a dispatcher. The Store can further have multiple Reducers, functions that act on and modify specific subsections of the Store.
For a more detailed discussion of Flux, I highly recommend Facebook's truly awesome official documentation: 
<sup><a href="https://facebook.github.io/flux/docs/overview.html">Flux official documentation</a></sup>

##4. A basic component
Let's build a simple component that reads an API endpoint that returns a string, and renders it onto the page. 

###4.1 Directory Structure

Before we dive in, it is helpful to have a good picture of file structure. A good way to structure a React/Redux application is as follows:

     app
      |-- package.json
      |-- components
          |-- sample_component
              |-- components
                  |-- header.js
              |-- reducers
                  |-- index.js
                  |-- sub_reducer.js
              |-- actions.js
              |-- index.js
              |-- store.js
          |-- router.js
          |-- routes.js
Now let's jump in.  
##4.2 Code
First let's write our store:

    //app/components/sample_component/store.js
    import { createStore, applyMiddleware } from 'redux'
    import thunk from 'redux-thunk'
    
    const createStoreWithMiddleware = applyMiddleware(thunk)(createStore)
    
    export default (rootReducer) =>
      createStoreWithMiddleware(rootReducer)

Now let's write our actions:

    //app/components/sample_component/actions.js
    import { resolve } from 'url'
    import request from 'http-as-promised'
    import Immutable from 'immutable'
    
    const defaultOptions = Immutable.Map({
      method: 'GET',
      json: true,
      resolve: 'body'
    })
    
    const makeRequest = (options) => {
      return request(defaultOptions.merge(options).toJS())
    }
    
    export const RECEIVE_TEXT = 'RECEIVE_TEXT'
    export function receiveText (text) {
       return (dispatch) => {
         dispatch({
           type: RECEIVE_TEXT,
           text
         })
       }
     }
    
    export function fetchText () {
      return (dispatch) => {
        return makeRequest({
          url: resolve('localhost:3000/', 'body/text')
        })
        .then(response => dispatch(receiveText(response.text)))
      }
    }

Let's wire these together with a reducer:

    //app/components/sample_component/reducers/index.js
    import { combineReducers } from 'redux'
    import bodyText from './body'
    
    export default function () {
      return combineReducers({
        bodyText
      })
    }

 And

    //app/components/sample_component/reducers/body.js
    import {
      RECEIVE_TEXT
    } from '../actions'
    
    const initialState = ''
    
    export default (state = initialState, action) => {
      switch (action.type) {
        case RECEIVE_TEXT:
          return action.text
        default:
          return state
      }
    }

Finally, lets connect everything with a basic component:

    //app/components/sample_component/index.js
    import React, { Component } from 'react'
    import store from './store'
    import reducer from'./reducers'
    import { fetchText } from './actions'
    
    export default class BasicTextDisplay extends Component {
      static displayName = 'BasicTextDisplay'
      constructor (props) {
        super(props)
        this.state = {
          store: store(reducer()),
          loaded: false
        }
      }
    
      componentDidMount () {
        const { store: { dispatch } } = this.state
        dispatch(fetchText()).then(() => this.setState({
          loaded: true,
          bodyText: this.state.store.getState().bodyText
        }))
        
        this.subscriber = this.state.store.subscribe(() => {
          this.setState({
            bodyText: this.state.store.getState().bodyText
          })
        })
      }
      
      componentWillUnmount () {
        this.subscriber()
      }
 
      render () {
        const { bodyText } = this.state
        return (
          <div className='body-text'>
            { bodyText }
          </div>
        )
      }
    }

Finally, at a higher level we might have something like (assuming for a second a healthy build pipeline):

    ReactDOM.render(<BasicBasicTextDisplay/>, document.getElementById('content')

It is possible and/or desirable to do this through a router based on URL, we'll come back to that later.

Let's walk through the above example.

##4.3 An explanation

###4.3.1 Redux
Redux exposes a few top level api methods:

`createStore` accepts a reducer and creates a store using that reducer
`applyMiddleware` does exactly what the name implies and applies a middleware to the store
`subscribe` allows a component to subscribe to store changes
`dispatch` allows an action to be dispatched
`combineReducers` allows us to combine multiple reducers into a single reducer like so:
`getState` gets the current value of the store

It is important to note, that `subscribe` returns a function, let's call it `subscriber`. This function, when run, will remove the listener from the store.

A full discussion of the Redux API is available at the: <sup><a href='http://rackt.org/redux/docs/api/index.html'>Redux Official Documentation</a></sup>

###4.3.2 ES2015
Object shorthand initialization allows us to shorten the creation of our objects, as long as the key and value are the same, eg:

    combineReducers({
       reducer1: reducer1,
       reducer2: reducer2
    })

  ES2015 allows us to shorten this to:

    combineReducers({reducer1, reducer2})

Or, in the case of our example,

      combineReducers({
        bodyText: bodyText
      })
      
is shortened to:

    combineReducers({bodyText})

Another syntactically interesting observation is the line:
  

    const { store: { dispatch } } = this.state

This is using the ES2015 object destructuring feature, and can be rewritten as:

      const store = this.state.store
      const dispatch = store.dispatch

Finally, this article has and will make copious use of ES2015's new `class` syntax, as well as `import` and `export`. As such, a detailed discussion is in order.

The first, and most important observation is that the `class` keyword does not follow the traditional inheritance model of the OO world. Instead, it is simply a wrapper around standard Javascript prototypical inheritance. Things defined as "methods" go on the prototype chain, while things defined on this directly go on the object itself. Like so:

    class Foo {
      constructor () {
        this.value = "Hello!"
      }
 
      printWord () {
        console.log("World!")
      }
    }

Then running:

    let foo = new Foo()
    foo.hasOwnProperty(value) // => true
    foo.hasOwnProperty(printWord) // => false
    foo.__proto__.hasOwnProperty(printWord) //=> true

While the `extends` keyword simply chains prototypes.

`import` and `export` are providing a module system. They can be loosely compared to the old requirejs system. `export default` provides the name for the `import` from a file. Anything exported without `default`, must be enumerated within curly braces like so:

    import { foo } from './foo'        
    
### 4.3.3 Redux Thunk

redux-thunk is a middleware library which allows actions to do preprocessing and allows us to treat synchronous and asynchronous actions in the same way.

So our store.js simply creates a redux store with redux-thunk middleware applied, and our root reducer simply wraps our body reducer.<sup><a href='https://github.com/gaearon/redux-thunk'>Redux Thunk Official Documentation</a></sup>

### 4.4.4 Logic

Now let's get to the meaty bits:
Our body reducer follows the default redux reducer structure. It is a function with a default value (in this case the empty string), and a switch statement that returns the default value on an unknown action, or a new value for this particular part of the store. It is crucial to note that the reducer *never* mutates the value, but instead generates a *new singleton value*. For the purpose of clarification, a call to `getState()` returns:

    {
      body: ''
    }

while we only currently have one reducer, we could have separate parts of the state tree, which would be modified by different reducers responding to actions.

Our actions.js is fairly self explanatory, enumerating the action symbols to which the store responds, and defining dispatchable functions (courtesy of redux-thunk) for asynchronous calls and/or preprocessing.

Finally our component, at instantiation time (in the constructor), creates its state. Once the component mounts (is rendered), `componentDidMount` will run, and the `fetchText()` action we defined is dispatched. Immediately thereafter, the component subscribes to the store to ensure that any changes are picked up.



It is worth pausing now, to talk about the React Component lifecycle

## 5. The React Component lifecycle
A component has the following lifecycle methods:

`constructor (props)`
`componentWillMount(props)`
`componentDidMount(props)`
`componentWillReceiveProps(props)`
`shouldComponentUpdate(props, state)`
`componentWillUpdate(props, state)`
`componentDidUpdate(props, state)`
`componenWillUnmount()`
`render()`
`setState()`

The constructor is where you should do all initial configuration of your component. This runs first

`componentWillMount` and `componentDidMount` both run once, directly before and after first render respectively
`componentWillReceiveProps` runs whenever a components props (passed as DOM attributes on the element) are changing
`shouldComponentUpdate` runs any time `setState` is called, or any time new `props` are received. By default, either of these actions will trigger `render`, `shouldComponentUpdate` lets you make this conditional. It is relatively rare that you will require this.
`componentWillUpdate` and `componentDidUpdate` will run before and after each rerender respectively
`componentWillUnmount` runs just before the component is removed from DOM
`render` is a function, the return value of which is a JSX representation of your component.
`setState` is how you should update a components state. Be acutely aware that directly modifying state like so:

    this.state.foo = 'newValue'

*WILL NOT* trigger your component to update. Instead, call `setState({foo: 'newValue'})`

A full discussion of the component Lifecyle  is available at the: <sup><a href='https://facebook.github.io/react/docs/component-specs.html'>Lifecycle Official Documentation</a></sup>

## 6. React Router and the basics of component composition
Probably the easiest way to have a single entry point into your application is to use React Router to manager Component Renders. This will also help you avoid Server refreshes as you can avoid wholesale pageloads. This also provides a wonderful entry into discussion how to compose multiple components into a single one as routing is virtually impossible without this.

Let's build our router:

    'use strict'
    
    import React from 'react'
    import { Router, browserHistory } from 'react-router'
    import { render } from 'react-dom'
    import routes from './routes'
    
    render(<Router history={browserHistory}>{routes}</Router>, document.getElementById('mainApp'))
    
And our routes:

    import React from 'react'
    import { TextBody, Container } from './components'
    import { Route, IndexRoute } from 'react-router'
    
    export default
      <Route path='/' component={Container}>
        <Route path='/textBody'>
          <IndexRoute component={TextBody} />
        </Route>
      </Route>
      
Lets build a Container component:

    'use strict'
    
    import React, { Component, PropTypes } from 'react'
    import Navigation from '../navigation'
    
    export default class extends Component {
      static displayName = 'Container';
      static propTypes = {
        children: PropTypes.node
      };
      render () {
        return (
          <div className='container'>
            <Navigation />
            <div id='content' className='content'>
              {this.props.children}
            </div>
          </div>
        )
      }
    }

Our Navigation Component:

    'use strict'
    
    import React, { Component } from 'react'
    import { Link } from 'react-router'
    
    export default class extends Component {
      static displayName = 'Navigation';
    
      render () {
        return (
          <ul>
            <li>
              <Link to='/textBody'>Text Body Demo</Link>
            </li>
          </ul>
        )
      }
    }
    
Our components `index.js` to pull it all together:

    'use strict'
    
    export TextBody from './text_body'
    export Container from './container'
    
Lets break this down:
Our router servers as our entry point, and creates a top level "god component" called `Container`.

This Component is built through composition of the `Navigation` component, importing one component into another to encapsulate functionality.

A centralized `components/index.js` file acts as a manifest, pulling together all components for consumption by the router using the `export ... from` feature of ES2015

Our router also passes all children on a `<Route component>` Component down to the component as `this.props.children`. In general, `this.props` is usually used to pass information between components, from the parent down, with only the top component in a given tree consuming the Redux store directly as state.  In general, props are set as element attributes, for example `<Route path='foo' component={Component}>`, the `this.props` of the `<Route>` component are `this.props.path` and `this.props.component`. It is generally recommended to validate your props as strictly as possible using propTypes as in the above example:

      static propTypes = {
        children: React.PropTypes.node
      };

While more examples will be present in this article, a full list of PropType validations is available at the: <sup><a href='https://facebook.github.io/react/docs/reusable-components.html'>PropTypes Official Documentation</a></sup>

## 7. Advanced composition, inheritance, and proptypes

That concludes our basic example. Lets add a more advanced example. Let's say we're building a multi tab display, with each pane showing a different content. Let's also presume, that for legacy reasons, our api returns the first tab's content as a string, the second tab's content as an array of words, and the third tab's content as on object of the form:


    {
      text: value
    }

    
Let's further say we wish to have a way to refresh our content, by, for example, clicking a button.

Let's finally say that we wish to have two different fundamental types of content, a textarea, or a div, represented by an editable value on the body of the tab contents.

A sample json might look something like:

    {
      tab1: {
        editable: false,
        body: 'LOREM IPSUM'
      },
      tab2: {
        editable: false,
        body: ['LOREM', 'IPSUM']
      },
      tab3: {
        editable: false,
        body: {text: 'LOREM IPSUM'}
      },
      tab4: {
        editable: true,
        body: 'LOREM IPSUM'
      },
      tab5: {
        editable: true,
        body: ['LOREM', 'IPSUM']
      },
      tab6: {
        editable: true,
        body: {text: 'LOREM IPSUM'}
      }
    }

How would we build this? There are multiple levels of complexity. First, we need to figure out a way to conditionally show certain subsections of data. Second, we need to figure out a way to use disparate logic depending on the shape of the body, third, we want to use different rendering logic depending on a flag, and finally, in the interests of keeping our code DRY, we want to do this in as few lines of code as possible.

We're not going to spend too long in the actions or the reducers or the store. Suffice it to say they look almost identical to our previous example at first, an action to fetch and action to update the store after fetching. Our component also looks almost identical to our basic component from before, setting up and subscribing to the store.
          
The one thing worth noting, is that to avoid mutation, and all of the bugs caused thereby, the object is immediatly cast to an Immutable as soon as the action reaches the reducer.

Lets dive into our components:

Our parent component might have the following render function:

      render () {
        return (
          <span>
            { this.renderTabSelector() }
            <div className='body-text'>
              { this.renderTabContents() }
            </div>
          </span>
        )
      }
      
With `renderTabSelector` and `renderTabContents` defined as follows:

      selectTab (key) {
        const { store: { dispatch } } = this.state
        dispatch(selectTab(key))
      }
    
      renderTabSelector () {
        if (this.state.tabs) {
          return this.state.tabs.keySeq().map((k) => {
            return <button onClick={this.selectTab.bind(this, k)} key={k}>{k}</button>
          })
        }
      }
    
      renderTabContents () {
        if (this.state.tabs && this.state.tabKey) {
          return <TabContentsRouter contents={this.state.tabs.get(this.state.tabKey)}/>
        }
      }

Our `TabContentsRouter` could look something like:

    'use strict'
    
    import React, { Component } from 'react'
    import ArrayBodyComponent from './array_body_component'
    import StringBodyComponent from './string_body_component'
    import ObjectBodyComponent from './object_body_component'
    
    export default class extends Component {
      static displayName = 'Body Router';
      static propTypes = {
        contents: (props) => {
          const { contents } = props
          if (!contents) {
            return new Error('Contents must be provided to the Tab Contents Router')
          }
          if (typeof (contents.editable) !== 'boolean') {
            return new Error('Contents must have an editable value of type boolean')
          }
          if (['string', 'object'].indexOf(typeof (contents.body)) === -1) {
            return new Error('Contents body must be either a string or a an object/array')
          }
        }
      };
    
      pickComponentType () {
        const { editable, body } = this.props.contents
        if (Array.isArray(body)) {
          return <ArrayBodyComponent body={body} editable={editable}/>
        } else if (typeof (body) === 'string') {
          return <StringBodyComponent body={body} editable={editable}/>
        } else {
          return <ObjectBodyComponent body={body} editable={editable}/>
        }
      }
    
      render () {
        return (<span>{ this.pickComponentType() }</span>)
      }
    }
    
Note the custom proptypes validator.

Finally let us define a superclass for the three component types to encapsulate the common functionality:

    import React, { Component, PropTypes } from 'react'
    
    export default class extends Component {
      static propTypes = {
        editable: PropTypes.bool.isRequired,
        body: PropTypes.oneOfType([
          PropTypes.string,
          PropTypes.shape({
            text: PropTypes.string
          }),
          PropTypes.arrayOf(PropTypes.string)
        ]).isRequired
      };
    
      renderTextArea (body) {
        return <textarea defaultValue={body}></textarea>
      }
    
      renderNonEditable (body) {
        return <div> {body} </div>
      }
    
      render () {
        if (this.props.editable) {
          return <span>{ this.renderTextArea(this.constructBody()) }</span>
        } else {
          return <span>{ this.renderNonEditable(this.constructBody()) }</span>
        }
      }
    }
    
Now lets extend down from this. In the interests of space, only one will be shown, although all code will be provided:
    
    import { PropTypes } from 'react'
    import MasterTabContentComponent from './master_tab_content'
    
    export default class extends MasterTabContentComponent {
      static propTypes = {
        body: PropTypes.shape({
          text: PropTypes.string
        }).isRequired
      };
    
      constructBody () {
        return this.props.body.text
      }
    }

There are three things worthy of note here. First, the TabRouterComponent, uses Component composition to pull in multiple other components to render the tabs and bodies. Second the superclass (MasterTabComponent) does almost identical prop validations to the TabRouterComponent without using a custom validator function, checking for sum types and union types on the attributes.
Finally, the child component inherits from the parent, overriding part of the proptype check, and defines its subsections of the behaviour (construcBody) while repurposing most of the parent classes' behaviour wholesale.

## 8. The build chain
Building ES2015 into ES5 (what most browsers accept) is fairly trivial with the help of babel. Browserify lets us pull together npm packages as ui dependencies using `import` or `require`. Standard is a wonderful code style tool, which removes opinion from the equation, gulp is a task runner, and npm/node is the package manager and bundler.

Lets see how it's all connected:
  
    > npm install babelify --save-dev
    > npm install browserify --save-dev
    > npm install gulp --save-dev
    > npm install babel-preset-es2015 --save-dev
    > npm install babel-preset-stage-0 --save-dev
    > npm instlal babel-eslint --save-dev
    > npm install babel-preset-react --save-dev
    > npm install gulp --save-dev
    > npm install standard --save-dev
    
Generates all of our tooling for us. Now we can add the following into our `package.json`:

      "standard": {
        "parser": "babel-eslint"
      }
      
### 8.1 Standard
An aside about standard. I personally happen to disagree with some of standard's stylistic choices. For example, I find that the lack of semicolons in particular is awful, making code harder to reason about in a static reading, and making parenthetical groupings cause function executions. The below will for example, cause `foo` to execute with `(bar && bazz)` as arguments
    
    const x = foo
    (bar && bazz) ? bar : bazz
    
With that said, why do I still use standard and recommend its use. In short, due to bikeshedding. When engineers are given the freedom to argue about code formatting and linting, as jshint, jslint, and eslint all do with the ability to override or create custom rules, they inevitably spend an inordinate amount of time and energy arguing. With standard, that is taken out of the equation. The stylistic choices over the code are out of the control of *everybody*, letting engineers solve actual problems, not whether or not there should be two spaces or a tab.

### 8.2 gulp

    const bundle = function (bundler) {
      bundler
        .bundle()
        .pipe(source('build.js'))
        .pipe(gulp.dest('./build/'))
    }
    
    gulp.task('browserify', function () {
      const bundler = browserify(['./assets/router.js'], {fullPaths: true})
        .transform(babelify, {presets: ['es2015', 'react', 'stage-0']})
      bundle(bundler)
    })
    
What does the above do? It reads assets/router.js, and uses browserify to construct a single file by following all links (`import` and `require`) it then uses babel with the configured presets to transpile ES2016 into ES5. Finally it pipes that to ./build/build.js. Now all we need is to server build/build.js in the HTML template provided from the server.

## 9. Conclusion
This article has presented an approach to solve many of the problems inherent to client side development. It utilizes the latest and greatest language features of ES2015, while maintaining support for legacy browsers, allows us to lean on the rich ecosystem of npm packages, utilizes an immutable data framework to avoid referential bugs and global state, cleanly encapsulates the statefulness of client side development as actions,and does so in away that I believe is easy to reason about.In addition this article has outlined some best practices for react and / or redux to help you componentize and compose your code.