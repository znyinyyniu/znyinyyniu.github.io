---
layout: post
title: angular开发技巧
description: 
date: 2017-12-31
---

# ngOnChanges生命周期钩子

1. 当Angular（重新）设置数据绑定输入属性时响应。 该方法接受当前和上一属性值的SimpleChanges对象，当被绑定的输入属性的值发生变化时调用，首次调用一定会发生在ngOnInit()之前。

2. 开发一个页面提示组件，当页面中有提示信息需要展示时，就会弹出一个提示框，如下图所示：

    <img src="../../../assets/images/angular-dev-tips/pageTip.png">

    html
    ```
        <app-page-tip [msg]="pageTip.msg"></app-page-tip>
    ```

    ts
    ```ts
        pageTip: PageTip = new PageTip();

        this.pageTip.showSuccess('保存成功！');
    ```

    pageTip.ts
    ```ts
    export class PageTip {

        msg: Message = new Message();

        showSuccess(summary: string, detail?: string) {
            this.clear();
            this.msg = { severity: 'success', summary: summary, detail: detail ? detail : '' };
        }

        clear() {
            this.msg = new Message();
        }
    }

    export class Message {
        severity: string;
        summary: string;
        detail: string;
    }
    ```

3. 当app-page-tip组件的msg(即pageTip.msg)有内容时，页面自动弹出一个提示框，并显示msg。这就需要用到ngOnChanges生命周期钩子。

    page-tip.component.ts
    ```ts
    @Component({
        selector: 'app-page-tip',
        templateUrl: './page-tip.component.html',
        styleUrls: ['./page-tip.component.css']
    })
    export class PageTipComponent implements OnInit, OnChanges {

        @Input()
        msg: Message;

        ngOnChanges(changes: SimpleChanges): void {
            let previousMsg = <Message>changes.msg.previousValue;
            let currentMsg = <Message>changes.msg.currentValue;
            if (!currentMsg.summary) {
                return;
            }
            $('#' + this.id).dialog({
                resizable: false,
                width: "400px",
                height: "auto",
                modal: true,
                title: this.title
            });
        }
    }
    ```
    这里用到了jquery-ui中的dialog插件。

4. 参考资料：[angular生命周期钩子](https://www.angular.cn/guide/lifecycle-hooks)

# 让组件通过ngModel的形式实现双向绑定

1. 开发一个下拉框组件，其当前选中的项能通过 [(ngModel)]="XXX" 这样的形式实现双向绑定，如下图所示：

    <img src="../../../assets/images/angular-dev-tips/dropdown.png">

    html
    ```
        <app-dropdown width="100" height="30" [selectAreaHeight]=120 [items]="areas" [(ngModel)]="curArea"></app-dropdown>
    ```

    ts
    ```ts
        curArea: DropdwonItem;
    ```
2. 实现ControlValueAccessor接口，writeValue方法用于将model更新到视图，onModelChange方法用于将视图更新到model。将此组件作为NG_VALUE_ACCESSOR的提供者。

    dropdown.component.ts
    ```ts
    @Component({
        selector: 'app-dropdown',
        templateUrl: './dropdown.component.html',
        styleUrls: ['./dropdown.component.css'],
        providers: [{
            provide: NG_VALUE_ACCESSOR,
            useExisting: forwardRef(() => DropdownComponent),
            multi: true
        }]
    })
    export class DropdownComponent implements OnInit, ControlValueAccessor {

        @Input()
        items: DropdwonItem[];

        @Input() width: number;
        @Input() height: number;

        @Input() selectAreaHeight: number = 350;

        curDropdwonItem: DropdwonItem = new DropdwonItem();

        constructor() { }

        private onModelChange: Function = () => { };
        private onModelTouched: Function = () => { };

        writeValue(obj: any): void {
            if (obj) {
                this.curDropdwonItem = obj;
            }
        }

        registerOnChange(fn: any): void {
            this.onModelChange = fn;
        }
        registerOnTouched(fn: any): void {
            this.onModelTouched = fn;
        }

        switch(item: DropdwonItem) {
            this.isShowSelect = false;
            if (item.code === this.curDropdwonItem.code) {
                return;
            }
            this.curDropdwonItem = item;
            this.onModelChange(this.curDropdwonItem);
            this.onChanged.emit(this.curDropdwonItem);
        }
    }

    export class DropdwonItem {
        code: string;
        name: string;
    }
    ```

    dropdown.component.html
    ```
    <div *ngIf="styleType==1" [ngStyle]="{'width':width+'px'}" class="option_box" (mouseenter)="onMouseEnter()" (mouseleave)="onMouseLeave()">
        <a class="selected" href="javascript:void(0);" [ngStyle]="{'height':height+'px','line-height':height+'px'}">
             { {curDropdwonItem?.name} } 
            <i [ngStyle]="{'top':top}" [ngClass]="{'down': !isShowSelect,'up': isShowSelect}"></i>
        </a>
        <div [ngStyle]="{'top':listTop,'width':width+'px','max-height':selectAreaHeight+'px'}" class="select_more" *ngIf="isShowSelect">
            <a href="javascript:void(0);" (click)="switch(item)" *ngFor="let item of items">`{ {item.name} }
            </a>
        </div>
    </div>
    ```

3. 参考资料：[ControlValueAccessor](https://www.angular.cn/api/forms/ControlValueAccessor)

# 如何手动触发angular的变更检测

1. 所有的异步操作(Events、Timers、XHRs)都会触发变更检测。

2. 开发日期选择控件时，由于是包装jquery的WdatePicker插件，所以需要响应其onpicked、oncleared事件。由于这两个事件无法触发变更检测，导致日期选择后，视图没有作出相应的变化。

3. 可以在组件中注入ApplicationRef对象，然后调用其tick方法，这样就可以触发变更检测了。

    date-picker.component.ts
    ```ts
        constructor(private appRef: ApplicationRef) {
        }

        picker() {
            WdatePicker({
                dateFmt: this.dateFmt,
                el: this.dateId,
                minDate: this.minDate ? this.minDate : "",
                maxDate: this.maxDate ? this.maxDate : "",
                onpicked: () => {
                    this.curDate = $("#" + this.dateId).val();
                    this.onModelChange(this.curDate);
                    this.appRef.tick();
                },
                oncleared: () => {
                    this.curDate = $("#" + this.dateId).val();
                    this.onModelChange(this.curDate);
                    this.appRef.tick();
                }
            });
        }
    ```
4. 参考资料：[These 5 articles will make you an Angular Change Detection expert](https://blog.angularindepth.com/these-5-articles-will-make-you-an-angular-change-detection-expert-ed530d28930)

# MathJax渲染

1. 当试题的内容渲染到Dom后，需要触发MathJax将其中的latex渲染成公式。可以使用setTimeout来完成。

    ```ts
        setTimeout(() => {
            MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
        }, 10);
    ```

2. 参考资料：[JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)

# 如何为纯粹的attribute(没有对应的Dom属性)绑定值

1. 有一个div，想为它添加data-topicid属性，且此属性要绑定到数据模型，可以采用下面的方式实现，在属性前面加上“attr.”前缀。

    ```
    <div [attr.data-topicid]="topic.topicId"></div>
    ```

2. 参考资料：[attribute、class 和 style 绑定](https://www.angular.cn/guide/template-syntax#attribute、class-和-style-绑定)

# 如何将html内容完整的绑定到div中

1. 使用innerHTML属性绑定，不使用{ {    } }
2. 使用BypassSecurityTrustHtmlPipe

    bypass-security-trust-html.pipe.ts
    ```ts
    @Pipe({
        name: 'bypassSecurityTrustHtml'
    })
    export class BypassSecurityTrustHtmlPipe implements PipeTransform {

        constructor(private domSanitizer: DomSanitizer) {
        }

        transform(value: any, args?: any): any {
            return this.domSanitizer.bypassSecurityTrustHtml(value);
        }

    }
    ```

    html
    ```
    <div [innerHTML]="paperHtml | bypassSecurityTrustHtml"></div>
    ```

3. 参考资料：[信任安全值](https://www.angular.cn/guide/security#bypass-security-apis)

# 如何在子组件中调用父组件的属性或方法

1. 有时候，我们需要在子组件中调用父组件的属性或方法。这也是可以作到的。

2. 在子件中通过@Inject与父组件的类型注入父组件

    ```ts
        constructor(private commonService: CommonService,
            private router: Router,
            private errorTopicService: ErrorTopicService,
            private sessionStorageService: SessionStorageService,
            @Inject(IndexComponent) private parent: IndexComponent
        ) { }
    ```

3. 在子组件中调用父组件的方法

    ```ts
        openSimulateDialog() {
            this.parent.nav.openSimulateDialog();
        }

        closeSimulateDialog() {
            this.parent.nav.closeSimulateDialog();
        }
    ```

# 异步路由

1. 一个angular应用体积会比较大，如果首次需要全部加载，则体验很不好。可以采用分模块的方式，根据路由地址按需异步加载。

    app-routing.module.ts
    ```ts
    @NgModule({
        imports: [
            RouterModule.forRoot(
                [
                    { path: 'index', loadChildren: './index/index.module#IndexModule' },
                    { path: 'special', loadChildren: './special/special.module#SpecialModule' },
                    { path: 'mylib', loadChildren: './mylib/mylib.module#MyLibModule' },
                    { path: 'qualityLib', loadChildren: './qualityLib/qualityLib.module#QualityLibModule' },
                    { path: 'manualGroup', loadChildren: './manual-group/manualGroup.module#ManualGroupModule' },
                    { path: 'schoolLib', loadChildren: './school/school.module#SchoolModule' },
                    { path: 'other', loadChildren: './other/other.module#OtherModule' },
                    { path: 'smartGroup', loadChildren: './smart-paper/smartPaper.module#SmartPaperModule' },
                    { path: 'demo', loadChildren: './demo/demo.module#DemoModule' }
                ]
            )
        ],
        exports: [RouterModule],
        providers: [SetTitleGuard]
    })
    export class AppRoutingModule {
    }
    ```

    index.module.ts
    ```ts
    @NgModule({
        imports: [
            CommonModule,
            ComModule,
            IndexRoutingModule,
            SelfComModule
        ],
        declarations: [
            IndexComponent,
            SurveyComponent,
            EntranceComponent,
            HotPaperComponent,
            ResourceStatsComponent
        ],
        providers: [
            IndexService,
            ErrorTopicService
        ],
    })
    export class IndexModule { }
    ```

2. 参考资料：[异步路由](https://www.angular.cn/guide/router#里程碑6：异步路由)

# 使用jquery动画、排序插件

一些动画、拖动排序等效果，如果全部采用angular重新实现，会比较耗时。此时可以直接使用原生的jquery插件。

# 目录结构

    ```
        assets
            css
                site.css（全局的智学网所有web应用都能复用的样式）
                common.css（选题组卷公用的样式）
        app
            common（全局的智学网所有web应用都能复用的组件、管道、工具类、服务等）
            self-common（选题组卷公用的组件、服务等）
            index（首页模块）
            mylib（我的卷库模块）
            other（错误等其他杂项）
    ```

# 开发环境跨域请求如何解决

1. 使用webpack-dev-server的代理功能

    proxy.conf.json
    ```json
    {
        "/api": {
            "target": "http://localhost:8080/paperfresh",
            "secure": false
        },
        "/assets/editor/mathml2internal": {
            "target": "http://localhost:8080",
            "pathRewrite": { "^/assets": "/paperfresh" },
            "secure": false
        },

        "/": {
            "target": "http://localhost:8080/paperfresh",
            "secure": false
        }
    }
    ```

    package.json
    ```json
    "scripts": {
        "ng": "ng",
        "start": "ng serve --proxy-config proxy.conf.json",
        "build": "ng build --prod --base-href ./",
        "copy": "gulp",
        "clean": "gulp clean",
        "test": "echo \"Error: no test... minimal project\" && exit 1",
        "lint": "echo \"Error: no lint... minimal project\" && exit 1",
        "e2e": "echo \"Error: no e2e... minimal project\" && exit 1"
    },
    ```

2. 使用跨域请求过滤器

    CorsFilter.java
    ```java
    public class CorsFilter implements Filter {

        private String allowOrigin;
        private String allowMethods;
        private String allowCredentials;
        private String allowHeaders;
        private String exposeHeaders;
        
        private boolean isActive=false;

        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
            allowOrigin = filterConfig.getInitParameter("allowOrigin");
            allowMethods = filterConfig.getInitParameter("allowMethods");
            allowCredentials = filterConfig.getInitParameter("allowCredentials");
            allowHeaders = filterConfig.getInitParameter("allowHeaders");
            exposeHeaders = filterConfig.getInitParameter("exposeHeaders");
            
            Properties properties = new Properties();
            try {
                properties.load(CorsFilter.class.getClassLoader().getResourceAsStream("env.properties"));
            } catch (IOException e) {
                e.printStackTrace();
            }

            isActive = "true".equalsIgnoreCase(properties.getProperty("isCorsFilter"));
        }

        @Override
        public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
            if(!isActive){
                chain.doFilter(req, res);
                return;
            }
            
            HttpServletResponse response = (HttpServletResponse) res;
            if (StringUtils.isNotEmpty(allowOrigin)) {
                response.setHeader("Access-Control-Allow-Origin", allowOrigin);
            }
            if (StringUtils.isNotEmpty(allowMethods)) {
                response.setHeader("Access-Control-Allow-Methods", allowMethods);
            }
            if (StringUtils.isNotEmpty(allowCredentials)) {
                response.setHeader("Access-Control-Allow-Credentials", allowCredentials);
            }
            if (StringUtils.isNotEmpty(allowHeaders)) {
                response.setHeader("Access-Control-Allow-Headers", allowHeaders);
            }
            if (StringUtils.isNotEmpty(exposeHeaders)) {
                response.setHeader("Access-Control-Expose-Headers", exposeHeaders);
            }
            chain.doFilter(req, res);
        }

        @Override
        public void destroy() {
        }
    }
    ```

    web.xml
    ```xml
    <filter>
        <filter-name>corsFilter</filter-name>
        <filter-class>com.zhixue.tms.common.utils.CorsFilter</filter-class>
        <init-param>
            <param-name>allowOrigin</param-name>
            <param-value>http://localhost:4200</param-value>
        </init-param>
        <init-param>
            <param-name>allowMethods</param-name>
            <param-value>GET,POST,PUT,DELETE,OPTIONS</param-value>
        </init-param>
        <init-param>
            <param-name>allowCredentials</param-name>
            <param-value>true</param-value>
        </init-param>
        <init-param>
            <param-name>allowHeaders</param-name>
            <param-value>x-requested-with,origin,content-type,accept,jwt,uid</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>corsFilter</filter-name>
        <url-pattern>/api/*</url-pattern>
    </filter-mapping>
    ```

    发送跨域ajax请求时，应设置withCredentials属性为true，以便能带上cookie信息。
    ```ts
    static initRequestOptionsArgs(): RequestOptionsArgs {
        if (environment.production) {
            return {};
        } else {
            return { withCredentials: true };
        }
    }

    save(): Promise<any> {
        let params = ComUtil.initURLSearchParams();
        return this.http.post(`${this.urlPrefix}/save`, params, ComUtil.initRequestOptionsArgs())
        .toPromise()
        .then(ComUtil.extractData)
        .catch(ComUtil.handleError);
    }

    getData(): Promise<any> {
        let params = ComUtil.initURLSearchParams();
        let requestOptionsArgs = ComUtil.initRequestOptionsArgs();
        requestOptionsArgs.search = params;
        return this.http.get(`${this.urlPrefix}/getData`, requestOptionsArgs)
        .toPromise()
        .then(ComUtil.extractData)
        .catch(ComUtil.handleError);
    }
    ```

# 引进angular的三种方式

1. 全新的web应用，直接使用angular。
2. 老的功能与框架不动，新老框架并存，新的功能用angular实现，老的功能在老的框架中维护。逐步等待机会将老的功能也用angular重新实现。
3. 将老的功能全部采用angular实现。