# 用户系统完善

## 完善登录功能
我们之前设计的是点击右上角的头像就会进行登录，那么说明右上角的头像应该是链接到`github`授权的页面，这个我们就直接来做：
```javascript
// components/layout.jsx
import { useState, useCallback } from 'react'
import { Layout, Icon, Input, Avatar,Tooltip,Dropdown, Menu } from 'antd' // 1. 引入Tooltip、Dropdown、Menu这三个组件
import Container from './Container.jsx'

const {Header, Content, Footer} = Layout

import { connect } from 'react-redux'  // 2. 引入redux，因为要从redux中读取用户信息显示在页面上
import getConfig from 'next/config'  // 3.1 引入nextjs配置中的内容
const { publicRuntimeConfig } = getConfig() // 3.2 主要引入github授权的网址链接

const githubIconStyle = {
	color: 'white',
	fontSize: 40,
	display: 'block',
	paddingTop: 10,
	marginRight: 20
}

const footerStyle = {
	textAlign: 'center'
}

function MyLayout ({children, user}) { // 9. 拿到user信息
	const [search, setSearch] = useState('')

	const handleSearchChange = useCallback((event)=> {
		setSearch(event.target.value)
	}, [setSearch])

	const handleOnSearch = useCallback(() => {}, [])

  // 6. 退出的逻辑我们后面再实现
	const userDropDown = (
		<Menu>
			<Menu.Item>
				<a href="javascript:void(0)">
					登出
				</a>
			</Menu.Item>
		</Menu>
	)

	return (
		<Layout>
			<Header>
				<Container renderer={<div className="header-inner" />}>
					<div className="header-left">
						<div className="logo">
							<Icon type="github" style={githubIconStyle}></Icon>
						</div>
						<div>
							<Input.Search
								placeholder="搜索仓库"
								value={search}
								onChange={handleSearchChange}
								onSearch={handleOnSearch}
							></Input.Search>
						</div>
					</div>
					<div className="header-right">
						<div className="user">
							{
								user && user.id ? (
                  // 5. redux中有用户信息，说明已经登录，显示用户的头衔
									<Dropdown overlay={userDropDown}>
										<a href="/">
											<Avatar size={40} src={user.avatar_url}/> {/* 10. 从user信息中取到user的头像*/}
										</a>
									</Dropdown>
								) : (
                  // 4. redux中没有user信息就显示为登录，Tooltip是一个移动上去有提示的组件
									<Tooltip title="登录">
										<a href={publicRuntimeConfig.OAUTH_URL}>
											<Avatar size={40} icon="user"/>
										</a>
									</Tooltip>
								)
							}
						</div>
					</div>
				</Container>
			</Header>
			<Content>
				<Container>
					{children}
				</Container>
			</Content>
			<Footer style={footerStyle}>
				Develop by Taopoppy @<a href="mailto:taopoppy@63.com">taopoppy@63.com</a>
			</Footer>
			<style jsx>{`
				.header-inner {
					display: flex;
					justify-content: space-between;
				}
				.header-left {
					display: flex;
					justify-content: flex-start;
				}
			`}</style>
			<style jsx global>{`
				#__next {
					height: 100%;
				}
				.ant-layout {
					height: 100%;
				}
				.ant-layout-header {
					padding-left: 0;
					padding-right: 0;
				}
			`}</style>
		</Layout>
	)
}

// 8. 从rendux中拿到user的信息，以props的形式传入MyLayout组件中
const mapStateToProps = (state) => {
	return {
		user: state.user
	}
}

// 7. 引入redux
export default connect(mapStateToProps)(MyLayout)
```
当然了，我们在`Layout`当中去获取`store`中的数据，我们就必须在`pages/_app.js`当中去调换一下`Provider`和`Layout`的包裹顺序：
```javascript
// pages/_app.js
return (
  <Container>
    <Head>
      <title>Taopoppy</title>
    </Head>
    <Provider store={reduxStore}> {/* 1. Provider在外面 */}
      <Layout> {/* 2. Layout在里面 */}
          <Component {...pageProps}/>
      </Layout>
    </Provider>
  </Container>
)
```


## 用户登出功能
用户登出功能要有两个步骤：
+ <font color=#9400D3>首先在页面上我们要通过修改store中的数据来清空客户端store的中user模块中的数据</font>
+ <font color=#9400D3>然后我们需要通过POST请求到后端来清除保存在session当中的用户数据</font>

我们先来为之前登出的按钮添加点击事件：
```javascript
// components/layout.jsx
import { logout } from '../store/store.js'

function MyLayout ({children, user, logout}) { // 3. 引入logout函数
	...

	// 2. 在点击事件当中去执行props.logout函数
	const handleLogout = useCallback(() => {
		logout()
	}, [])

	const userDropDown = (
		<Menu>
			<Menu.Item>
				<a href="javascript:void(0)" onClick={handleLogout}> {/*1. 给登出添加点击事件*/}
					登出
				</a>
			</Menu.Item>
		</Menu>
	)

	return (
		...
	)
}

// 5. 在MyLayout.props.logout函数当中去dispatch一个名字叫做logout的异步action
const mapDispatchToProps = (dispatch) => {
	return {
		logout: () => {
			dispatch(logout())
		}
	}
}

// 4. 添加mapDispatchToProps
export default connect(mapStateToProps,mapDispatchToProps)(MyLayout)
```

然后我们去定义这个异步的`action`：
```javascript
// store/store.js
import axios from 'axios'

const userInitialState = {}
const LOGOUT = 'LOGOUT'  // 3. 定义个actionType

function userReducer(state = userInitialState,action) {
	switch (action.type) {
		// 4. 当action.type为LOGOUT的时候，我们清空store中的user模块的数据即可
		case LOGOUT: {
			return {}
		}
		default:
			return state
	}
}

// 1. 定义一个异步的action，名字叫做logout
export function logout() {
	return dispatch => {
		axios.post('/logout').then(resp => {
			if (resp.status === 200) {
				// 2. 请求成功，就dispatch一个对象类型的action，type为LOGOUT，value为空
				dispatch({
					type: LOGOUT
				})
			} else {
				console.log('logout failed', resp)
			}
		}).catch(err=> {
			console.log('logout failed catch', err)
		})
	}
}
```
然后页面上的事情做完后，我们来书写服务端的代码，应该只有清空了服务端的`session`，才算登出：
```javascript
// server/auth.js
module.exports = (server) => {
	...

  server.use( async (ctx, next)=> {
    const path = ctx.path
    const method = ctx.method
    if(path === '/logout' && method === 'POST') {
      ctx.session = null
      ctx.body = `logout success`
    } else {
      await next()
    }
  })
}
```


## 维持OAuth之前页面访问