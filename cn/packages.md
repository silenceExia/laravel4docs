# 包开发

- [简介](#introduction)
- [创建包](#creating-a-package)
- [包结构](#package-structure)
- [服务提供器](#service-providers)
- [包规定](#package-conventions)
- [开发流程](#development-workflow)
- [包路由](#package-routing)
- [包配置](#package-configuration)
- [包迁移](#package-migrations)
- [包Assets](#package-assets)
- [发布包](#publishing-packages)

<a name="introduction"></a>
## 简介

包是主要作用是为Laravel添加功能。包可以是任何东西，例如用于日期处理的[Carbon](https://github.com/briannesbitt/Carbon)，或者是一个完整的BDD测试框架[Behat](https://github.com/Behat/Behat)。

当然，还有很多不同类型的包。有些包是独立的，这意味着它们可以在任何框架中工作，而不仅仅是Laravel。上面提到的Carbon和Behat就是独立的包。要在Laravel中使用这些包只需要在`composer.json`文件中指明。

另一方面，有些包仅支持Laravel。在上一个Laravel版本中，这些类型的包我们称为"bundles"。这些包可以包含专为增强Laravel应用的路由、控制器、视图、配置和迁移。由于开发独立的包不需要专门的过程，因此，本手册主要涵盖针对Laravel开发独立的包。

所有Laravel包都是通过[Packagist](http://packagist.org)和[Composer](http://getcomposer.org)发布的，因此很有必要学习这些PHP包发布工具。

<a name="creating-a-package"></a>
## 创建包

为Laravel创建一个包的最简单方式是使用Artisan的`workbench`命令。首先，你需要在`app/confg/workbench.php`文件中配置一些参数。在该文件中，你会看到`name`和`email`两个参数，这些值是用来为新创建的包生成`composer.json`文件的。一旦你提供了这些值，就可以开始构建一个新包了！

**执行Artisan中的Workbench命令**

	php artisan workbench vendor/package --resources

厂商名称（vendor name）是用来区分不同作者构建的相同名字的包。例如，如果我（Taylor Otwell）创建了个名为"Zapper"的包，厂商名就可以叫做`Taylor`，包名可以叫做`Zapper`。默认情况下，workbench命令建的包是不依赖任何框架的；然而，`resources`命令将会告诉workbench创建特定于Laravel的一些目录，例如`migrations`、`views`、`config`等。

一旦执行了`workbench`命令，新创建的包就会出现在Laravel安装目录下的`workbench`目录中。接下来就应该为你创建的包注册`ServiceProvider`了。你可以通过在`app/config/app.php`文件里的`provides`数组中添加该包。这将通知Laravel在应用程序开始启动时加载该包。服务提供者（Service providers）使用`[Package]ServiceProvider`样式的命名方式。所以，以上案例中，你需要将`Taylor\Zapper\ZapperServiceProvider`添加到`providers`数组。

一旦注册了provider，你就可以开始写代码了！然而，在此之前，建议你查看以下部分来了解更多关于包结构和开发流程的知识。

<a name="package-structure"></a>
## 包结构

执行`workbench`命令之后，你的包将被初始化规定的结构，并能够与laravel框架融合：

**基本包目录结构**

	/src
		/Vendor
			/Package
				PackageServiceProvider.php
		/config
		/lang
		/migrations
		/views
	/tests
	/public

让我们来深入了解该结构。`src/Vendor/Package`目录是所有class的主目录，包括`ServiceProvider`。`config`、`lang`、`migrations`和`views`目录，就如你所猜测，包含了相应的资源。包可以包含这些资源中的任意几个，就像一个"regular"的应用。

<a name="service-providers"></a>
## 服务提供器

服务提供器只是包的引导类。默认情况下，他们包含两个方法：`boot`和`register`。你可以在这些方法内做任何事情，例如：包含路由文件、注册IoC容器的绑定、监听事件或者任何想做的事情。

`register`方法在服务提供器注册时被立即调用，而`boot`方法仅在请求被路由前调用。因此，如果服务提供器中的动作（action）依赖另一个已经注册的服务提供器，或者你正在覆盖另一个服务提供其绑定的服务，就应该使用`boot`方法。

当使用`workbench`命令创建包时，`boot`方法已经包含了如下的动作：

	$this->package('vendor/package');

该方法告诉Laravel如何为应用程序加载视图、配置或其他资源。通常情况下，你没有必要改变这行代码，因为它会根据workbench的默认约定将包设置好的。

By default, after registering a package, its resources will be available using the "package" half of `vendor/package`. However, you may pass a second argument into the `package` method to override this behavior. For example:

	// Passing custom namespace to package method
	$this->package('vendor/package', 'custom-namespace');

	// Package resources now accessed via custom-namespace
	$view = View::make('custom-namespace::foo');

There is not a "default location" for service provider classes. You may put them anywhere you like, perhaps organizing them in a `Providers` namespace within your `app` directory. The file may be placed anywhere, as long as Composer's [auto-loading facilities](http://getcomposer.org/doc/01-basic-usage.md#autoloading) know how to load the class.

<a name="package-conventions"></a>
## 包约定

要使用包中的资源，例如配置或视图，需要用双冒号语法：

**加载包中的视图**

	return View::make('package::view.name');

**获取包的某个配置项**

	return Config::get('package::group.option');

> **注意：** 如果你包中包含迁移，请为迁移名（migration name）添加包名作为前缀，以避免与其他包中的类名冲突。

<a name="development-workflow"></a>
## 开发流程

当开发一个包时，能够使用应用程序上文是相当有用的，这样将允许你很容易的解决视图模板的等问题。所以，我们开始，安装一个全新的Laravel框架，使用`workbench`命令创建包结构。

在使用`workbench`命令创建包后。`workbench/[vendor]/[package]`目录使用`git init`，`git push`！这将允许你在应用程序中方便开发而不用为`composer update`命令苦扰。

当包存放在`workbench`目录时，你可能担心Composer如何知道自动加载包文件。当`workbench`目录存在，Laravel将智能扫描该目录，在应用程序开始时加载它们的Composer自动加载文件！

<a name="package-routing"></a>
## 包路由

在之前的Laravel版本中，`handlers`用来指定那个URI包会响应。然而，在Laravel4中，一个包可以相应任意URI。要在包中加载路由文件，只需在服务提供器的`boot`方法`include`它。

**在服务提供器中包含路由文件**

	public function boot()
	{
		$this->package('vendor/package');

		include __DIR__.'/../../routes.php';
	}

<a name="package-configuration"></a>
## 包配置

有时创建的包可能会需要配置文件。这些配置文件应该和应用程序配置文件相同方法定义。并且，当使用 `$this->package`方法来注册服务提供器时，那么就可以使用“双冒号”语法来访问：

**访问包配置文件**

	Config::get('package::file.option');

然而，如果你包仅有一个配置文件，你可以简单命名为`config.php`。当你这么做时，你可以直接访问该配置项，而不需要特别指明文件名：

**访问包单一配置文件**

	Config::get('package::option');

### 级联配置文件

但其他开发者安装你的包时，他们也许需要覆盖一些配置项。然而，如果从包源代码中改变值，他们将会在下次使用Composer更新包时又被覆盖。替代方法是使用artisan命令 `config:publish`：

**执行配置公布命令**

	php artisan config:publish vendor/package

当执行该命令，配置文件就会拷贝到`app/config/packages/vendor/package`，开发者就可以安全的更改配置项了。

> **注意:** 开发者也可以为该包创建指定环境的配置文件通过替换配置项并放置在`app/config/packages/vendor/package/environment.

<a name="package-migrations"></a>
## 包迁移

你可以很容易在包中创建和运行迁移。要为工作台里的包创建迁移，使用`--bench`选项：

**为工作台的包创建迁移**

	php artisan migrate:make create_users_table --bench="vendor/package"

**为工作台包运行迁移**

	php artisan migrate --bench="vendor/package"

要为已经通过Composer安装在`vendor`目录下的包执行迁移，你可以直接使用`--package`：

**为已安装的包执行迁移**

	php artisan migrate --package="vendor/package"

<a name="package-assets"></a>
## 包Assets

有些包可能含有assets，例如JavaScript，CSS，和图片。然而，我们无法链接到`vendor`或`workbench`目录里的assets，所以我们需要可以将这些assets移入应用程序的`public`目录。`asset:publish`命令可以实现：

**将包Assets移动到Public**

	php artisan asset:publish

	php artisan asset:publish vendor/package

If the package is still in the `workbench`, use the `--bench` directive:

	php artisan asset:publish --bench="vendor/package"

This command will move the assets into the `public/packages` directory according to the vendor and package name. So, a package named `userscape/kudos` would have its assets moved to `public/packages/userscape/kudos`. Using this asset publishing convention allows you to safely code asset paths in your package's views.

<a name="publishing-packages"></a>
## 发布包

当你创建的包准备发布时，你应该将包提交到[Packagist](http://packagist.org)仓库。如果你的包只针对Laravel，最好在包的`composer.json`文件中添加`laravel`标签。

还有，在发布的版本中添加tag，以便开发者能当请求你的包在他们`composer.json`文件中依赖稳定版本。如果稳定版本还没有好，考虑直接在Composer中使用`branch-alias`。

一旦你的包发布，舒心的继续开发通过`workbench`创建的应用程序上下文。这是很方便的方法来在发布包后继续开发包。

一些组织使用他们私有分支包为他们自己开发者。如果你对这感兴趣，查看Composer团队构建的[Satis](http://github.com/composer/satis)文档。

译者：王赛  [（Bootstrap中文网）](http://www.bootcss.com)