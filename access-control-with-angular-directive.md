# 使用Angular指令+服务实现权限控制
公司已有的代理商管理系统，权限控制需要调整，我负责前端相关工作，此文记录一下修改过程。  

## 原有实现
前端原有的权限控制是基于角色的，即根据当前登录账号的角色，控制菜单及功能入口的显示/隐藏状态。前端主要用到框架Angular和Angular-ui-router，在这前提下，原有的权限控制实现方式大致如下：

> 在`mainController`中获取当前登录用户的角色，并将其赋值到`$rootScope.role`，页面中根据该变量做基于角色的权限控制。具体实现即使用`ng-show="role === ?"`控制元素显示/隐藏，达到不同角色显示不同内容的效果。

上述实现的最大问题在于前端权限控制严重依赖系统固有角色，极度僵硬，角色变动会给前端带来巨大冲击。原有的丑陋实现之所以能够存在，跟技术人员的水平有一定关系，同时也因为系统初期相对简单，只包含3个角色：root、agent、user，才没有暴露出问题。随系统功能增加，角色与权限都需要再细分，按照之前的实现方式，很难维护，不得已才重新设计与实现前端的权限控制。  

同时由于历史原因，存在这样一种情况：

> 本是相同性质的接口，却按角色不同，拆分成了不同接口。比如日志查询，root调用的是`/logs/findAllLogs`，agent调用的是`/logs/findAgentLogs`。

这是系统设计时犯的一个错误，分权分域，域和权的混淆。为处理上述不同角色调用不同接口的情况，在其他`controller`中也经常也需要使用变量`$rootScope.role`，因此在这些`controller`中依赖角色信息的代码，需要等待`mainController`中角色信息获取成功后方可执行。  

## 新方案
要解决权限控制僵硬的问题，核心就是前端权限控制不应该依赖角色，而是依赖登录账号所拥有的权限项，这需要后台做两个修改：
1.   提供接口获取当前登录账号的权限项。
2.   因分权分域混淆，导致同性质接口出现多个的情况，需要将多个接口合并为一个接口。

后台配合修改后，接下来考虑前端实现，实现方式完全可以参照原有实现，只需要将角色概念替换成权限项。在`mainController`中获取账号权限项，赋值给`$rootScope.permissions`，提供是否拥有权限的判断方法`$rootScope.permissionExists = function (item) {...};`。剩下的都是体力活，删除页面上跟`role`相关的逻辑，再按照新的方式（`ng-show="permissionExists(?)"`）为页面上的功能绑定权限。  

虽然参照原有方式实现没啥明显缺陷，但自己对于Angular不熟悉，基于探索和学习的精神，打算使用自定义指令与服务达到相同效果，同时了解如何实现Angular自定义指令与服务。

## 指令(Directives)的实现过程
使用指令，最终设想的效果与`ng-show="permissionExists(permission)"`类似，提供一个指令`xsAccess`，页面使用`xs-access="pId"`的方式为UI元素绑定所需权限，其中`pId`表示具体权限项标识符，xs则是命名空间约束，避免冲突。  

由于Angular正常注册指令只能在初始化阶段同步注册，为达到上述效果，所面临的第一个问题：

> 由于权限项是通过Ajax异步获取的，在注册指令时账号拥有的权限项并不确定，Angular渲染页面也先于Ajax返回，因此，在获取权限项的过程中，相关UI元素的的显示/隐藏状态也就无法确定。  

为解决上述问题，采用如下方式(代码参见文章底部)：

> 在数据获取过程中，将指令的link方法缓存，数据获取完成时，再调用一次link方法。

上述方式虽能保持异步获取权限项成功后，保证页面上权限相关元素状态正确，但更好的方式可能是使用[运行时动态注册指令](http://stackoverflow.com/questions/31556913/dynamically-register-directive-at-runtime)，下次有机会再实践。  

## 使用服务(Services)
使用指令的方式绑定权限已经实现，但还需考虑随登陆状态变化权限项也要重新获取，主要原因如下：

> 基于Angular-ui-router实现的单页面应用，在用户不刷新页面的情况下，不同`state`间跳转，应用不会重新初始化。在第一次初始化，xsAccess内部获取并持有用户权限项信息，而当用户在不同`state`间跳转时，用户权限可能发生变化，如`logout => login`、`login => logout`。当这种情况发生，xsAccess内部维护的权限项信息不再有效，此时界面跟权限相关元素的状态也可能错误。

为解决上述问题，考虑了两种方案：
1.   在指令内部注册监听事件，在需要更新权限项的地方触发该事件，达到通知更新目的。
2.   注册一个服务，在需要更新权限项的地方，调用服务提供的方法，达到通知更新目的。

第一种方案，指令竟然对外提供事件触发机制去保持状态正确，直觉上很怪异，用过的一些指令也没见这样搞事的，所以选择第二种看起来科学一点的方式。注册一个xsAccess的服务，对外提供`fresh`方法用于刷新。同时需要考虑权限获取过程中，相关UI元素的的显示/隐藏状态无法确定，在权限获取完成时及时更新这些UI元素的状态。  

最终，也就是注册一个指令用于界面权限绑定，以及注册一个服务维护当前账号权限项信息。由于项目已使用RequireJs做模块化，指令和服务直接通过局部变量共享信息，省略部分细节，主要代码如下：

```javascript
// 项目使用RequireJs做模块化，app即应用module
// 事后认为单独注册一个module，然后在service上暴露两个方法(init、fresh)更合理
define([], function () {
    var accessList = [];
    var fetching = true;
    var dirties = [];

    function init(app) {
        app.directive('xsAccess', function () {
            return {
                restrict: 'A',
                scope: null,
                link: function (scope, element, attrs) {
                    if (fetching) {
                        var that = this;
                        dirties.push(function () {
                            that.link.call(that, scope, element, attrs);
                        });
                        element.hide();
                        return;
                    }

                    _elementStatus(element, attrs);
                }
            };

            function _elementStatus(element, attrs) {
                // todo 根据元素所需权限显示/隐藏
            }
        }).factory('xsAccess', ['$http', function ($http) {
            fetchPermissions();

            return {
                fresh: fetchPermissions
            };

            function fetchPermissions() {
                fetching = true;

                var url = '/svc/userPermission';
                $http.get(url).success(function (res) {
                    accessList = res;

                    fetching = false;
                    dirties.forEach(function () {
                        item.call(null);
                    });
                    dirties = [];
                }).error(function (err) {
                    // todo error handling
                });
            }
        }]);
    }

    return {
        init: init
    };
});
```