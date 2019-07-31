---
title: "Don’t use Laravel’s config() inside config files"
date: 2016-12-19T18:00:00+07:00
tags: ["tech"]
language: "english"
---

_Originally published at [blog.maqe.com](https://blog.maqe.com/dont-use-laravel-s-config-inside-config-files-40e2c8207225)_

_a.k.a. Underlying OS behavior can impact your web application’s behavior._

_Update: As of 2017–10–12, config file loading in Laravel 5.5 is now ordered by file names. See the pull request [[5.5] Ensure config load order across multiple installations.](https://github.com/laravel/framework/pull/21634)_

{{<figure src="resources/01-laravel-load-configuration.png" title="A code excerpt from Laravel’s Illuminate\Foundation\Bootstrap\LoadConfiguration">}}

Last week we were working on implementing a feature for a Laravel project running on load-balanced EC2 instances. The feature that we built required two configuration values which normally would reside happily within a file or two in the `config/` folder.

However, this time our team tried something a bit differently. We had a config that references another config in another file, like this:

**config/a.php:**

```php
<?php
return [
    'a_value' => 'my_config_value',
];
```

**config/b.php:**

```php
<?php
return [
    'b_value' => config('a.a_value'),
];
```

## What happened?

Once we deployed the code to our test environment, we noticed one strange behavior. One instance was able to lookup the config values successfully. However, the other returned `null`.

**Instance 1:**

```shell
$ php artisan tinker
>>> config('a.a_value');
=> "my_config_value"
>>> config('b.b_value');
=> "my_config_value"
```

**Instance 2:**

```shell
$ php artisan tinker
>>> config('a.a_value');
=> "my_config_value"
>>> config('b.b_value');
=> null
```

We checked deployment logs and file content on both servers. The code was deployed successfully with identical instructions, and the files were identical. Basically everything looked identical on two instances but the results were different!

## The solution

After tinkering with `config()`, it turned out that using `config()` outside config files seemed to work fine. Only using Laravel’s `config()` in another config file that does not always yield the correct value. **Hence, never use Laravel’s `config()` in config files.**

Although it was tempting to stop the investigation just there, we knew we could do more…

## The search for enlightenment

We made our servers provisioning and code deployment fully scripted and automated just so that we would avoid problems like this! Two instances that produce different results defeats the whole purpose of having a fully automated workflow.

The root cause must be found.

Since the server instances were more or less identical, we believed we could trace the code back far enough to find the point where the code’s output starts to diverge. It must be working identically up to some point, and right after that point should be the key to the root cause. So here is the journey down that road.

## How config() gets its values?

During bootstrap, Laravel does the following:

- Creates a `Illuminate\Config\Repository` class as storage for all config values.
- Load all configuration files, loop all configuration files and put all the configs found into the `Repository` above.
- When a code calls `config()`, it retrieves value by calling `Repository::get()`. Laravel simply returns the value that has already been loaded during bootstrap.

### But really, how config() gets its values?

So now we know that all config values are already loaded from their files right at bootstrap. We dig further to see how that works in Laravel’s `src/Illuminate/Foundation/LoadConfiguration.php`:

In `LoadConfiguration.php`, it loops through each configuration file one by one:

```php
<?php
protected function loadConfigurationFiles(...)
{
    foreach ($this->getConfigurationFiles($app) as $key => $path) {
        $repository->set($key, require $path);
    }
}
```

The code above gets the file list from `getConfigurationFiles()`:

```php
<?php
use Symfony\Component\Finder\Finder;

...

protected function getConfigurationFiles(Application $app)
{
    ...
    foreach (Finder::create()->files()->name('*.php')->in($configPath) as $file) {
    ...
    }
}
```

The code above uses Symfony’s `Symfony\Component\Finder\Finder` to get the file list. And so we wrote some tinker code to see if the two server instances produce the same result:

```php
$ php artisan tinker

use Symfony\Component\Finder\Finder;

foreach (Finder::create()->files()->name('*.php')->in('config/') as $file) {
    echo $file->getFilename();
}
```

**Result — Instance 1:**

```shell
"session.php"
"domains.php"
"urls.php"
"auth.php"
"app.php"
```

**Result — Instance 2:**

```shell
"urls.php"
"auth.php"
"domains.php"
"session.php"
"app.php"
```

Dumping the returns from `Finder` above, we see dissimilar results for the two server instances. Digging down even further reveals that Symfony uses PHP’s `RecursiveDirectoryIterator`. This behavior was reported and concluded in [RecursiveDirectoryIterator output is not sorted](https://bugs.php.net/bug.php?id=70734) as follow:

> The manual does not guarantee anything about the order.
>
> RecursiveDirectoryIterator (and the other filesystem based iterators) iterate in the order the OS returns the files to them.
>
> That it is sorted on Windows is not guaranteed by PHP, it just seems to happen due to the underlying implementation in the Windows kernel.

By investigating further, we found the inode numbers of the config files that confirms this:

**Instance 1:**

```shell
$ ls -i1 config/ | sort -n
309925 session.php
309926 domains.php
309932 urls.php
309940 auth.php
309941 app.php
```

**Instance 2:**

```shell
$ ls -i1 config/ | sort -n
324900 urls.php
324905 auth.php
324906 domains.php
324914 session.php
324915 app.php
```

The files sorted by inode numbers exactly matched the sequence that `RecursiveDirectoryIterator` returned. This means that on Instance 1, `urls.php` is be able to see configurations from `domains.php`, but Instance 2 does not, which exactly matches the behavior we found at the beginning.

## Lessons learnt:

- **As a Laravel’er:** Don’t use Laravel’s `config()` inside config files. The file system may produce varying results on different machines and/or operating systems.
- **As a coder:** Don’t assume your system will behave consistently just because you have identical code. If you expect a behavior, be explicit about it. If you need files sorted by file names, then explicitly sort them, don’t assume just because ls or your IDE tells you so.
- **As a DevOps/QA:** This proves how important it is to have a test environment that mimics production environment as close as possible. You can’t call your web applications scalable if your testing does not involve an comparably scalable test environment.

---

This writing is the result of effort put in by a team at MAQE Bangkok Co., Ltd. consisting of Siravit “Arm” Praditkul, Atthaphon “Jo” Urairat, Wanida “Gift” Sittipanya and Panupong “Big” Sritananun.

---

Bonus point: What’s an inode? An inode [contains essentially information about ownership (user, group), access mode (read, write, execute permissions) and file type of a file.](https://unix.stackexchange.com/questions/4402/what-is-a-superblock-inode-dentry-and-a-file/4403#4403)
