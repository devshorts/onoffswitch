---
layout: post
title: Typed react native router
date: 2019-08-05 21:20:36.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- dependency injection
- frontend
- react
meta:
  _su_rich_snippet_type: none
  _edit_last: '1'

permalink: "/2019/08/05/typed-react-native-router/"
---
<!-- wp:quote -->

> Note, a seasoned UI friend of mine told me this is a terrible way to do things and to instead use hooks instead

<!-- /wp:quote -->

<!-- wp:paragraph -->

I decided to give react native a go the last few days, just to try something different. For what it's worth I haven't done any real front-end work in quite a bit. The last stuff I did was angular 2/(8/a million?) and it was internal dev tooling stuff that was kind of hacky.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Seems like the javascript world has come quite a distance since back when typescript was still in beta!

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

That said, I dove into a sample react native app that I scaffolded out using [expo](https://expo.io/). Then I popped openn IntelliJ and started to play with react-native-paper and react-navigation.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

The first thing I wanted to see how to do properly was to do DI, since in my mind without clearly having DI you can't build large scale applications. There's a lot of ways to do DI but you need to make sure to not couple your code to its dependencies. Seems like React's answer to that is props, contexts, and redux. I haven't looked much at redux yet, but my initial reaction to props and context was lackluster. Props smells like a giant property god object to pass around and that never ends well.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

However, typescript does make this simpler and I do kind of like the higher order container stuff.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

After making a little hello world app and playing with the react router, I wanted to see if we could type the router. For example, in all the examples I can see you use

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
this.props.navigation.navigate("Home")
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

To navigate around. This screams bad design to me because I can tell already people will start sprinkling in static strings everywhere, running into typos and other hard to track down problems. What I would rather have is

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
this.navigation.home()
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Which is typed, discoverable, and easy to re-use.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

However, it took me quite a bit to figure out how to inject

<!-- /wp:paragraph -->

<!-- wp:list {"ordered":true} -->

1. Capture an instance of props
2. Wrap the navigation object into an object we can type
3. Pass that object to a component

<!-- /wp:list -->

<!-- wp:paragraph -->

Turns out higher order containers were needed and the magic sauce is here. First I can make my routable class that given an instance of the screen router can do the things it wants to do.

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
type RoutableProps = { navigation: NavigationScreenProp\<any, any\> } class Routable { private navigation: NavigationScreenProp\<any, any\>; constructor(navigation: NavigationScreenProp\<any, any\>) { this.navigation = navigation; } home() { console.log("home") this.navigation.navigate("Home") } back() { console.log("back") this.navigation.goBack() } profile() { console.log("profile") this.navigation.navigate("Profile") } }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Assuming we have this (somehow) we can use it in our nav bar

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
interface NavProps { router: Routable } export class NavBar extends React.Component\<NavProps\> { render() { const navigate = this.props.router; return ( \<Appbar\> \<Appbar.BackAction onPress={() =\> navigate.back()}/\> \<Appbar.Action icon="archive" onPress={() =\> navigate.home()}/\> \<Appbar.Action icon="mail" onPress={() =\> navigate.profile()}/\> \</Appbar\> ); } }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Now here we can wrap a higher order container to capture the props, inject the setting we want, and return a new decorated component

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
function withRouter(Component) { // inject the navigation route to the props of the component return withNavigation( class extends React.Component\<RoutableProps\> { render() { const nav = this.props.navigation; // give the component a router return ( \<Component router={new Routable(nav)}/\> ); } } ); } export const Nav = withRouter(NavBar);
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

Now we can safely use our typed navigation component

<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code {"language":"jscript"} -->

```
export default function Home() { return ( \<PaperProvider theme={theme}\> \<SafeAreaView style={styles.bottom}\> \<Nav/\> \<Title\>Hello\</Title\> \</SafeAreaView\> \</PaperProvider\> ); }
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->

And whamo! Typed navigation.

<!-- /wp:paragraph -->

<!-- wp:paragraph -->

Full source available at my github [https://github.com/devshorts/react-native-samples](https://github.com/devshorts/react-native-samples)

<!-- /wp:paragraph -->

