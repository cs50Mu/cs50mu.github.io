+++
title = "Grafana源码分析"
date = "2021-02-05"
slug = "2021/02/05/sourcecode-of-grafana"
Categories = []
+++

## Grafana源码分析

### 服务自动注入 / inject

代码里的service使用了一个包来实现自动注入：[`facebookarchive / inject`](https://github.com/facebookarchive/inject)

要实现这个自动，需要做以下准备工作：

打struct tag

```go
type HTTPServer struct {
      log           log.Logger
      macaron       *macaron.Macaron
      context       context.Context
      streamManager *live.StreamManager
      httpSrv       *http.Server

      RouteRegister        routing.RouteRegister            `inject:""`
      Bus                  bus.Bus                          `inject:""`
      RenderService        rendering.Service                `inject:""`
      Cfg                  *setting.Cfg                     `inject:""`
      HooksService         *hooks.HooksService              `inject:""`
      CacheService         *localcache.CacheService         `inject:""`
      DatasourceCache      datasources.CacheService         `inject:""`
      AuthTokenService     models.UserTokenService          `inject:""`
      QuotaService         *quota.QuotaService              `inject:""`
      RemoteCacheService   *remotecache.RemoteCache         `inject:""`
      ProvisioningService  provisioning.ProvisioningService `inject:""`
      Login                *login.LoginService              `inject:""`
      License              models.Licensing                 `inject:""`
      BackendPluginManager backendplugin.Manager            `inject:""`
      PluginManager        *plugins.PluginManager           `inject:""`
      SearchService        *search.SearchService            `inject:""`
  }
```

启动server时会build Service Graph:

```go
// 获取注册的服务，具体实现比较简单，分析略过
services := registry.GetServices()
if err = s.buildServiceGraph(services); err != nil {
  return
}


// 看看它做了啥呢，这里就是关键了
  // buildServiceGraph builds a graph of services and their dependencies.
  func (s *Server) buildServiceGraph(services []*registry.Descriptor) error {
      // Specify service dependencies.
      objs := []interface{}{
          bus.GetBus(),
          s.cfg,
          routing.NewRouteRegister(middleware.RequestMetrics, middleware.RequestTracing),
          localcache.New(5*time.Minute, 10*time.Minute),
          s,
      }

      for _, service := range services {
          objs = append(objs, service.Instance)
      }

      var serviceGraph inject.Graph

      // Provide services and their dependencies to the graph.
      for _, obj := range objs {
          // 奥秘在这里了
          if err := serviceGraph.Provide(&inject.Object{Value: obj}); err != nil {
              return errutil.Wrapf(err, "Failed to provide object to the graph")
          }
      }

      // Resolve services and their dependencies.
      if err := serviceGraph.Populate(); err != nil {
          return errutil.Wrapf(err, "Failed to populate service dependency")
      }

      return nil
  }
```

### user login是如何实现的？

有个middleware，在处理业务请求之前会先验证user

```go
// pkg/middleware/middleware.go
func GetContextHandler(
	ats models.UserTokenService,
	remoteCache *remotecache.RemoteCache,
	renderService rendering.Service,
) macaron.Handler {
	return func(c *macaron.Context) {
		ctx := &models.ReqContext{
			Context:        c,
			SignedInUser:   &models.SignedInUser{},
			IsSignedIn:     false,
			AllowAnonymous: false,
			SkipCache:      false,
			Logger:         log.New("context"),
		}

		orgId := int64(0)
		orgIdHeader := ctx.Req.Header.Get("X-Grafana-Org-Id")
		if orgIdHeader != "" {
			orgId, _ = strconv.ParseInt(orgIdHeader, 10, 64)
		}

		// the order in which these are tested are important
		// look for api key in Authorization header first
		// then init session and look for userId in session
		// then look for api key in session (special case for render calls via api)
		// then test if anonymous access is enabled
		// 这个写法可以学习一下
		switch {
		case initContextWithRenderAuth(ctx, renderService):
		case initContextWithApiKey(ctx):
		case initContextWithBasicAuth(ctx, orgId):
		case initContextWithAuthProxy(remoteCache, ctx, orgId):
		case initContextWithToken(ats, ctx, orgId):
		case initContextWithAnonymousUser(ctx):
		}

		ctx.Logger = log.New("context", "userId", ctx.UserId, "orgId", ctx.OrgId, "uname", ctx.Login)
		ctx.Data["ctx"] = ctx

		c.Map(ctx)

		// update last seen every 5min
		if ctx.ShouldUpdateLastSeenAt() {
			ctx.Logger.Debug("Updating last user_seen_at", "user_id", ctx.UserId)
			if err := bus.Dispatch(&models.UpdateUserLastSeenAtCommand{UserId: ctx.UserId}); err != nil {
				ctx.Logger.Error("Failed to update last_seen_at", "error", err)
			}
		}
	}
}
```

```go
// login接口的实现，它会调用下面的`AuthenticateUser`方法
// pkg/api/login.go
func (hs *HTTPServer) LoginPost(c *models.ReqContext, cmd dtos.LoginCommand) Response {

// 这里的逻辑包括：验证user、写cookie（middleware.WriteSessionCookie）
}


// pkg/login/auth.go
// AuthenticateUser authenticates the user via username & password
func AuthenticateUser(query *models.LoginUserQuery) error {
	if err := validateLoginAttempts(query.Username); err != nil {
		return err
	}

	if err := validatePasswordSet(query.Password); err != nil {
		return err
	}

	err := loginUsingGrafanaDB(query)
	if err == nil || (err != models.ErrUserNotFound && err != ErrInvalidCredentials && err != ErrUserDisabled) {
		return err
	}

	ldapEnabled, ldapErr := loginUsingLDAP(query)
	if ldapEnabled {
		if ldapErr == nil || ldapErr != ldap.ErrInvalidCredentials {
			return ldapErr
		}

		if err != ErrUserDisabled || ldapErr != ldap.ErrInvalidCredentials {
			err = ldapErr
		}
	}

	if err == ErrInvalidCredentials || err == ldap.ErrInvalidCredentials {
		if err := saveInvalidLoginAttempt(query); err != nil {
			loginLogger.Error("Failed to save invalid login attempt", "err", err)
		}

		return ErrInvalidCredentials
	}

	if err == models.ErrUserNotFound {
		return ErrInvalidCredentials
	}

	return err
}
```

### 总线机制是如何工作的？

到处都看到`bus.Dispatch`，那么bus是如何知道action对应的handler的呢？一定有一个register的机制，但没找到呢


```go
// 这样注册
bus.AddHandler("sql", UpdateUserLastSeenAt)

// 这里是handler的实现
func UpdateUserLastSeenAt(cmd *models.UpdateUserLastSeenAtCommand) error {
	return inTransaction(func(sess *DBSession) error {
		user := models.User{
			Id:         cmd.UserId,
			LastSeenAt: time.Now(),
		}

		_, err := sess.ID(cmd.UserId).Update(&user)
		return err
	})
}

// AddHandler的实现
func (b *InProcBus) AddHandler(handler HandlerFunc) {
	handlerType := reflect.TypeOf(handler)
	queryTypeName := handlerType.In(0).Elem().Name()
	// 原来所谓注册就是一个map，key是handler的入参，val是handler
	b.handlers[queryTypeName] = handler
}

// Dispatch的实现
func Dispatch(msg Msg) error {
	return globalBus.Dispatch(msg)
}

func (b *InProcBus) DispatchCtx(ctx context.Context, msg Msg) error {
	var msgName = reflect.TypeOf(msg).Elem().Name()

	var handler = b.handlersWithCtx[msgName]
	if handler == nil {
		return ErrHandlerNotFound
	}

	var params = []reflect.Value{}
	params = append(params, reflect.ValueOf(ctx))
	params = append(params, reflect.ValueOf(msg))
    
    // 反射来调用
	ret := reflect.ValueOf(handler).Call(params)
	err := ret[0].Interface()
	if err == nil {
		return nil
	}
	return err.(error)
}
```

### setting是如何初始化的？

看到使用的时候是直接这样：

```go
if !setting.BasicAuthEnabled {
    return false
}
```

### dto的运用

```go
// pkg/api/dashboard.go
      dashItem := &dashboards.SaveDashboardDTO{
          Dashboard: dash,
          Message:   cmd.Message,
          OrgId:     c.OrgId,
          User:      c.SignedInUser,
          Overwrite: cmd.Overwrite,
      }

      dashboard, err := dashboards.NewService().SaveDashboard(dashItem, allowUiUpdate)
      if err != nil {
          return dashboardSaveErrorToApiResponse(err)
      }
```

### 业务error到api error的转换

```go
      dashboard, err := dashboards.NewService().SaveDashboard(dashItem, allowUiUpdate)
      if err != nil {
          return dashboardSaveErrorToApiResponse(err)
      }
```
```go
func dashboardSaveErrorToApiResponse(err error) Response {
	if err == models.ErrDashboardTitleEmpty ||
		err == models.ErrDashboardWithSameNameAsFolder ||
		err == models.ErrDashboardFolderWithSameNameAsDashboard ||
		err == models.ErrDashboardTypeMismatch ||
		err == models.ErrDashboardInvalidUid ||
		err == models.ErrDashboardUidToLong ||
		err == models.ErrDashboardWithSameUIDExists ||
		err == models.ErrFolderNotFound ||
		err == models.ErrDashboardFolderCannotHaveParent ||
		err == models.ErrDashboardFolderNameExists ||
		err == models.ErrDashboardRefreshIntervalTooShort ||
		err == models.ErrDashboardCannotSaveProvisionedDashboard {
		return Error(400, err.Error(), nil)
	}

	if err == models.ErrDashboardUpdateAccessDenied {
		return Error(403, err.Error(), err)
	}

	if validationErr, ok := err.(alerting.ValidationError); ok {
		return Error(422, validationErr.Error(), nil)
	}

	if err == models.ErrDashboardWithSameNameInFolderExists {
		return JSON(412, util.DynMap{"status": "name-exists", "message": err.Error()})
	}

	if err == models.ErrDashboardVersionMismatch {
		return JSON(412, util.DynMap{"status": "version-mismatch", "message": err.Error()})
	}

	if pluginErr, ok := err.(models.UpdatePluginDashboardError); ok {
		message := "The dashboard belongs to plugin " + pluginErr.PluginId + "."
		// look up plugin name
		if pluginDef, exist := plugins.Plugins[pluginErr.PluginId]; exist {
			message = "The dashboard belongs to plugin " + pluginDef.Name + "."
		}
		return JSON(412, util.DynMap{"status": "plugin-dashboard", "message": message})
	}

	if err == models.ErrDashboardNotFound {
		return JSON(404, util.DynMap{"status": "not-found", "message": err.Error()})
	}

	return Error(500, "Failed to save dashboard", err)
}
```
