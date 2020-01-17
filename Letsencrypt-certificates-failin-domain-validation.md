#  Let‘s Encrypt certificates fail in domain validation

[TOC]

# 一、 环境

#### CentOS 7.6 或者以上

按照 [Omnibus GitLab Docs](https://docs.gitlab.com/omnibus/README.html#installation-and-configuration-using-omnibus-package) 指导文件安装必须包

```bash
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld

sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix

curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash

sudo EXTERNAL_URL="https://git.demo.com" yum install -y gitlab-ce

```

#### 域名

git.demo.com 显然这个域名是内部解析，非FQDN不能被 Public DNS 访问，因此也不能被[letsencrypt](https://letsencrypt.org/zh-cn/docs/challenge-types/) 验证，这个会在后面说明。

我在/etc/hosts 配置了该记录

同时设置 ```sudo hostnamectl  set-hostname git.demo.com``` 



# 二、问题

#### 描述

gitlab 安装到最后会出现如下错误

```bash
Running handlers:
There was an error running gitlab-ctl reconfigure:

letsencrypt_certificate[git.demo.com] (letsencrypt::http_authorization line 5) had an error: RuntimeError: acme_certificate[staging] (/opt/gitlab/embedded/cookbooks/cache/cookbooks/letsencrypt/resources/certificate.rb line 25) had an error: RuntimeError: ruby_block[create certificate for git.demo.com] (/opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb line 108) had an error: RuntimeError: [git.demo.com] Validation failed, unable to request certificate

Running handlers complete
Chef Client failed. 5 resources updated in 23 seconds
```

详细的安装错误日志参考 [安装错误摘要](#安装错误摘要)



> 仔细查找更前面的控制台信息，其中代码部分太长进行了缩减，摘要内容如下，判断出现在无法获取认证凭据recipes/http_authorization 上，具体可以参考[Let's Encrypt](https://letsencrypt.org/zh-cn/docs/challenge-types/) 其中解释
>
> HTTP-01 验证方式
> 这是当今最常见的验证方式。Let’s Encrypt 向您的 ACME 客户端提供一个令牌，然后您的 ACME 客户端将在您对 Web 服务器的 http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>（用提供的令牌替换 <TOKEN>）路径上放置指定文件。该文件包含令牌以及帐户密钥的指纹。一旦您的 ACME 客户端告诉 Let’s Encrypt 文件已准备就绪，Let’s Encrypt 会尝试获取它（可能从多个地点进行多次尝试）。如果我们的验证机制在您的 Web 服务器上找到了放置于正确地点的正确文件，则该验证被视为成功，您可以继续申请颁发证书。如果验证检查失败，您将不得不再次使用新证书重新申请。
>
> 我们的 HTTP-01 验证最多接受 10 次重定向。我们只接受目标为“http:”或“https:”且端口为 80 或 443 的重定向。我们不目标为 IP 地址的重定向。当被重定向到 HTTPS 链接时，我们不会验证证书是否有效（因为验证的目的是申请有效证书，所以它可能会遇到自签名或过期的证书）。
>
> HTTP-01 验证只能使用 80 端口。因为允许客户端指定任意端口会降低安全性，所以 ACME 标准已禁止此行为。

#### 分析

gitlab安装包中nginx root根目录默认在/var/opt/gitlab/nginx/www，所以我们结合这两方面信息在如下目录寻找

`/var/opt/gitlab/nginx/www/.well-known/acme-challenge `

此时没有发现相应的文件（安装过程中会产生这个文件，安装失败会删除这个文件），详见 [认证错误摘要](#认证错误摘要)



我们的域名是私有，Let's Encrypt 认证服务无法通过这个域名访问到https://git.demo.com 下的这个文件无法Validate我们的域名有效性。

*另外即使域名是FQDN有效的， 也需要做对应的NATD port 映射，Let's Encrypt访问到内网的服务。*



# 三、解决方案

结合上面问题分析，我们采取如下方式



#### 方案A：无需关心

虽然出现这个错误，但是访问https://git.demo.com 正确进入gitlab，不会影响web访问（是否影响git ssh 方式未作验证， 推断也是不会有影响的)，只是每次gitlab-ctl reconfigure 会有这个错误提示。

极致的用户需要看后面的方案 :-)



#### 方案B：退回http方式

这个在这里就不多介绍，需要做如下步骤

```bash
rm -rf /etc/gitlab/ssl                # 当然你可以备份后在删除

external_url 'http://git.demo.com'    # vi /etc/gitlab/gitlab.rb 修改为不启用TLS
nginx['redirect_http_to_https'] = true
nginx['redirect_http_to_https_port'] = 80

sudo gitlab-ctl reconfigure           # bash运行
```



#### 方案C：放弃使用Let's Encrypt Validate方式

该方式适用于个人搭建家用服务，不会出现reconfigure的错误

```
letsencrypt['enable'] = false      # vi /etc/gitlab/gitlab.rb 修改为不启用TLS

sudo gitlab-ctl reconfigure        # bash运行     
```

其实方案C 等于对附录中提到的certificate.rb 部分代码不运行

vi /opt/gitlab/embedded/cookbooks/letsencrypt/resources/certificate.rb

comment source code from line 25 to 38

```ruby
	 25   acme_certificate 'staging' do
     26     alt_names new_resource.alt_names unless new_resource.alt_names.empty?
     27     key_size new_resource.key_size unless new_resource.key_size.nil?
     28     group new_resource.group unless new_resource.group.nil?
     29     owner new_resource.owner unless new_resource.owner.nil?
     30     chain "#{new_resource.chain}-staging" unless new_resource.chain.nil?
     31     contact contact_info
     32     crt "#{new_resource.crt}-staging"
     33     cn new_resource.cn
     34     key "#{new_resource.key}-staging"
     35     dir 'https://acme-staging-v02.api.letsencrypt.org/directory'
     36     wwwroot new_resource.wwwroot
     37     sensitive true
     38   end
```



#### 方案**D**：申请FQDN域名

注册一个FQDN域名（当然不能选择上面的实验域名）并且公开这个域名的web服务，一般情况下就不会碰到这个问题。

如果这种场景下遇到无法Validate 则考虑显示配置认证文件。

```bash
letsencrypt['enable'] = true 
letsencrypt['auto_renew'] = true 
letsencrypt['auto_renew_hour'] = 0 
letsencrypt['auto_renew_minute'] = 30 
letsencrypt['auto_renew_day_of_month'] = "*/4" 
nginx['custom_gitlab_server_config'] = "location /.well-known/acme-challenge/ {\n root /var/opt/gitlab/nginx/www/; \n}\n"
```

# 参考

https://forum.gitlab.com/t/letsencrypt-certificates-fail-in-domain-validation/18112
https://stackoverflow.com/questions/59239820/gitlab-letsencrypt-http-authorization-error
https://letsencrypt.org/docs/challenge-types/

# 附录

### 安装错误摘要


```bash
ruby_block[create certificate for git.demo.com] action run

        ================================================================================
        Error executing action `run` on resource 'ruby_block[create certificate for git.demo.com]'
        ================================================================================
       /opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb:111:in `block (3 levels) in class_from_file'
        # Declared in /opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb:108:in `block in class_from_file'
    
        ruby_block("create certificate for git.demo.com") do
          action [:run]
          default_guard_interpreter :default
          declared_type :ruby_block
          cookbook_name "letsencrypt"
          block #<Proc:0x00000000059dcb78@/opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb:109>
          block_name "create certificate for git.demo.com"
        end
    
      ================================================================================
      Error executing action `create` on resource 'acme_certificate[staging]'
      ================================================================================
    
      RuntimeError
      ------------
      ruby_block[create certificate for git.demo.com] (/opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb line 108) had an error: RuntimeError: [git.demo.com] Validation failed, unable to request certificate

Resource Declaration:
---------------------
# In /opt/gitlab/embedded/cookbooks/cache/cookbooks/letsencrypt/recipes/http_authorization.rb

  5: letsencrypt_certificate site do
  6:   crt node['gitlab']['nginx']['ssl_certificate']
  7:   key node['gitlab']['nginx']['ssl_certificate_key']
  8:   notifies :run, "execute[reload nginx]", :immediate
  9:   notifies :run, 'ruby_block[display_le_message]'
 10:   only_if { omnibus_helper.service_up?('nginx') }
 11: end

System Info:
------------
chef_version=14.13.11
platform=centos
platform_version=7.7.1908
ruby=ruby 2.6.3p62 (2019-04-16 revision 67580) [x86_64-linux]
program_name=/opt/gitlab/embedded/bin/chef-client
executable=/opt/gitlab/embedded/bin/chef-client
```

### 认证错误摘要


    file[/var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY] action create
    create new file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY
            date content in file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY from none to 67a341
           --- /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY    2020-01-17 03:56:56.414651155 +0800
    
    ​       +++ /var/opt/gitlab/nginx/www/.well-known/acme-challenge/.chef-tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY20200117-14268-1e3v9ux        2020-01-17 03:56:56.414651155 +0800
    
    ​       @@ -1 +1,2 @@
    
    ​       +tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY._lV8BpO6qTwNOerUfgtfCchYdn8cqRBECoCTrVCnCjs
    
    ​    
    
       - change mode from '' to '0644'
    
         hange owner from '' to 'root'
    
            - change group from '' to 'root'
    
              estore selinux security context
    
    ​    
    
       * file[git.demo.com SSL key] action nothing (skipped due to action :nothing)
    
          directory[/var/opt/gitlab/nginx/www/.well-known/acme-challenge] action nothing (skipped due to action :nothing)
    
            * file[/var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY] action nothing (skipped due to action :nothing)
    
               file[/var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY] action delete
    
    ​     
    
       - delete file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY
    
       * ruby_block[create certificate for git.demo.com] action run	create new file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY
    
    date content in file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY from none to 67a341


    --- /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY    2020-01-17 03:56:56.414651155 +0800
    
       +++ /var/opt/gitlab/nginx/www/.well-known/acme-challenge/.chef-tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY20200117-14268-1e3v9ux        2020-01-17 03:56:56.414651155 +0800
    
       @@ -1 +1,2 @@
    
       +tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY._lV8BpO6qTwNOerUfgtfCchYdn8cqRBECoCTrVCnCjs


       - change mode from '' to '0644'
    
       - change owner from '' to 'root'
    
       - change group from '' to 'root'
    
       - restore selinux security context

     * file[git.demo.com SSL key] action nothing (skipped due to action :nothing)
    
     * directory[/var/opt/gitlab/nginx/www/.well-known/acme-challenge] action nothing (skipped due to action :nothing)
    
     * file[/var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY] action nothing (skipped due to action :nothing)
    
    * file[/var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY] action delete
    - delete file /var/opt/gitlab/nginx/www/.well-known/acme-challenge/tIqSYzS1vFP_Y4brcFAMHgYP48HSKOLhWocnBIqb0AY
    * ruby_block[create certificate for git.demo.com] action run
