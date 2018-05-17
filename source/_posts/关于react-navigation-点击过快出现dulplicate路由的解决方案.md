---
title: 关于react-navigation 点击过快出现dulplicate路由的解决方案
date: 2018-05-17 10:34:39
tags: [react native, react navigation]
---

最近做的几个react native 项目都用到了 react navigation 这个路由框架。react navigation 这个框架有着堪称原生性能的路由框架在发布不久就基本把RN其他框架给比下去了。它功能非常强大，可配置性很高，虽然一开始上手会有点困难但熟练后简直是路由神器。不过最近我发现一个问题，当你点击跳转路由按键过快会导致重复弹出相同的路由，这简直不能忍。这算不算点击穿透？令我惊讶的是react navigation 官方团队竟然到现在还没解决这个问题。。。

我后来在网上看到不少人都遇到这个问题，主要有以下2种解决方案：

1. 通过debounce方式，即通过setTimeout方法，当用户点击跳转路由按钮就给点击事件设置一个setTimeout事件，通过一个状态位把这个按钮的响应在接下来几秒内屏蔽掉。  
    ```
        this.state.disabled = true
        setTimeout(() => {  
            this.state.disabled = false)  
        }, 2000)
    ```
    这种方法需要在所以点击事件注入这段代码，既不优雅且维护成本高。没有采用这种方案。  


2. 如果你的项目引用了redux 并集成了 react navigation， 可以用下面这种方案：  

    按照react navigation官方集成教学，你的路由的reducer应该是这个样子的：
    ```
    import Router from '../../components/Router'
    const navReducer = (state, action) => {
        const newState = Router.router.getStateForAction(action, state);
        return newState || state;
    }
    export default navReducer
    ```
    我们要改动的，就是这个reducer。其实就是覆写react navigation关于路由state计算的逻辑。在默认情况下，我们用了react navigation 提供的Router.router.getStateForAction 方法去根据当前state和action算出下一步的state。这里state的变化会对应我们的路由栈。如果我们想要防止上面说的多次点击出现重复的路由，就需要在计算路由变化时，判断下一步action的跳转路由是不是跟当前一致，如果是则直接返回当前state，否则就使用Router.router.getStateForAction 计算state并返回。代码如下：
    ```
    import Router from '../../components/Router'

    function _getCurrentRoute(navState) {
        if (navState.hasOwnProperty('index')) {
            return _getCurrentRoute(navState.routes[navState.index])
        }
        return navState
    }

    export const reducer = (state, action) => {
        const { type, routeName } = action
        if (type.startsWith('Navigation/')) {
            const lastRoute = _getCurrentRoute(state)
            if (routeName === lastRoute.routeName) {
                return state
            }
        }
        const newState = Router.router.getStateForAction(action, state)
        if (newState) {
            newState.currentRoute = _getCurrentRoute(newState)
        }
        return newState || state
    }
    ```
    这里通过一个_getCurrentRoute递归方法去找到当前的route，我算出来同时还把它放到了state里面的currentRoute，这样在别的地方也能通过redux 注入currentRoute获取到当前的route信息了。