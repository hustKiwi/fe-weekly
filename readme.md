### 确定基础依赖
- requirejs的AMD（异步模块依赖）
- coffee & stylus
- backbone & backbone layoutmanager
- api & proxy

### 规划目录结构
```shell
src_root
├── package.json
├── bower.json
├── coffeelint.json
├── nobone_sync_cfg.coffee
├── Cakefile
├── tpl/
│   ├── index.tpl.html
│   ├── bg.tpl.html
│   ├── layout.tpl.html
├── static/
    ├── js/
    │   ├── main.coffee
    │   ├── core/
    │   │   ├── cfg.coffee
    │   │   ├── backbone_init.coffee
    │   │   ├── utils.coffee
    │   ├── module/
    │   │   ├── channels.coffee
    │   ├── view/
    │   │   ├── channels.coffee
    │   ├── lib/
    │   │   ├── bower/
    │   │   ├── require.js
    ├── css/
    │   ├── main.styl
    │   ├── core/
    │   │   ├── base.styl
    │   │   ├── const.styl
    │   │   ├── layout.styl
    │   │   ├── mixin.styl
    │   ├── view/
    │   │   ├── channels.styl
    │   ├── ui/
    │   │   ├── pager.styl 
    ├── img/
```

### 构建编译方法
- 基于 `nobone` 构建 coffee & stylus renderer
- 使用cake构建命令行任务
```shell
➜  mbox git:(new-fm) cake
Cakefile defines the following tasks:

cake setup                # Setup
cake sync                 # Run nobone sync
cake proxy                # Run API proxy
cake dev                  # Run dev tools

      --sync-off     Disable the nobone sync.
      --proxy-off    Disable proxy.

➜  mbox git:(new-fm) cake dev
[2015-01-09 10:02:42] >> Listen: 8813 0ms
[2015-01-09 10:02:43] Watched: 1990 0ms
```

### 跑通基础页面
重在验证编译和配置，能得到从数据获取到页面渲染的hello world级页面即可。

### 代码重构优化
#### 提炼通用方法
```coffee
utils =
    api: (url, data, options) ->
        def = $.Deferred()
        defaults =
            api_url: '/dev/api/?tn={{method}}'
            type: 'GET'
            data_type: 'json'
            processData: false
            handle_err: (r) ->
                console?.error(r)
        opts = _.merge defaults, options

        getUrl = (method) ->
            return method if method.startsWith('/')
            opts.api_url.replace('{{method}}', method)

        if _.isArray(url)
            exec = _.map(url, (url) ->
                utils.api(url)
            )
            return $.when.apply(utils, exec)

        if _.isObject(url)
            options = data
            data = url.data
            url = url.url

        data or= {}
        data.hashcode = App.hash_code or ''

        $.ajax
            url: getUrl(url)
            data: data or {}
            dataType: opts.data_type
            type: opts.type

            success: (r) ->
                if r.errorCode is 22000
                    return def.resolve(r.data)

                { status, hash_code } = r

                if status is 0
                    App.hash_code = hash_code if hash_code
                    return def.resolve(r)

                opts.handle_err(r)
                def.reject(r)

            error: ->
                def.reject('ajax error')
                opts.handle_err('ajax error')

        def.promise()

    fetch_tmpl: (name, done) ->
        url = 'tmpl/' + name
        def = $.Deferred()
        JST = window.JST or= {}

        done = done or def.resolve
        if JST[url]
            done(JST[url])
        else
            require [url], (tmpl) ->
                window.JST[url] = tmpl
                done(tmpl)

        def.promise()
```
```stylus
icon($icon, $width, $height = $width)
    inline-block()
    size($width, $height)
    text-indent: -999em
    overflow: hidden
    background: url($img_path + $icon + '.png') no-repeat 0 0
    pngfix($icon)
```

#### 抽象复用结构
```
Backbone.Layout.configure({
    manage: true
    fetchTemplate: (name) ->
        utils.fetch_tmpl(name, @async())
})

mixins = do ->
    fetch = (args) ->
        def = $.Deferred()
        done = (r...) =>
            r = r[0] if r.length is 1
            if @_format
                r = @_format(r)
            def.resolve(r)
        fail = ->
            def.reject()

        if @_api
            utils.api(_.merge @_api, args).done(done).fail(fail)

        def.promise()
    ->
        @fetch = fetch

mixins.call Model::
mixins.call Collection::

_.extend View::, {
    initialize: ->
        if @model
            @listenTo(@model, 'change', @render)
}
```

#### 总结优秀实践
```
_.extend App, {
    create_module: (options) ->
        $.extend(true, {
            Models: {}
            Views: {}
        }, options)

    init_module: (name, options) ->
        defaults =
            args: []
            need_css: true
        opts = _.merge defaults, options

        deps = [
            "coffee/module/#{name}"
        ]
        if opts.need_css
            deps.push "css!styl/view/#{name}"

        require deps, (mod) ->
            mod.initialize.apply(null, opts.args)
}
```
```
define [
    'coffee/view/channels'
], (Views) ->
    Module = App.create_module({
        Views: Views
    })

    Module.Models.Main = Main = Backbone.Model.extend({
        _api:
            url: 'channellist'

        _format: (data) ->
            group_channels = _.groupBy data.channel_list, (c) ->
                c.cate_order

            channels = []
            cate_order = _.sortBy(_.keys group_channels)

            for cate in cate_order
                c = group_channels[cate]
                channels.push(_.sortBy c, (item) ->
                    item.pv_order
                )

            {
                cur_type: fmListModel.getCurType()
                channels: channels
            }
    })

    Module.initialize = ->
        main = new Main()
        mainView = new Views.Main({
            model: main
        })

        App.use_layout().then ->
            App.layout.setView('', mainView)
            main.fetch().done (r) ->
                main.set(r)

    Module
```

#### 配置化、工具化、自动化
例如：通过 `yeoman` 构建脚手架脚本；实现工具解决代码同步、合图等问题（ `nobone-sync` 和 `imerge` ）

#### 与业务解构，形成独立框架 / 工具
