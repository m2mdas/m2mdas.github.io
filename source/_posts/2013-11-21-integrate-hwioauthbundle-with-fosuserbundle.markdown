---
layout: post
title: "Integrate HWIOAuthBundle with FOSUserBundle"
date: 2013-11-21 20:22
comments: true
categories: [PHP, Symfony, FOSUserBundle, HWIOAuthBundle]
description: Integrate HWIOAuthBundle with FOSUserBundle
keywords: PHP, Symfony2, FOSUserBundle, HWIOAuthBundle, configure FOSUserBundle and HWIOAuthBundle, setup, integrate
---
[HWIOAuthBundle](https://github.com/hwi/HWIOAuthBundle) is a great Symfony2 bundle that provides way to integrate web services that implements OAuth1.0 and OAuth2 as user authentication system. Once configured you can add infinite amount of web services as authentication source.

After user authentication it is better to fetch user information from the web service and store them in DB so that the user does not have to input profile information again. In following section I will outline step by step instruction on how to configure `HWIOAuthBundle` and integrate `FOSUserBundle` user provider using `fosub_bridge` implemented in `HWIOauthBundle`. For web service Github OAuth api used.

`HWIOAuthBundle` uses [Buzz](https://github.com/kriswallsmith/Buzz) curl client to communicate with web services. `Buzz` by default enables SSL certificate check. On some server CA certificate information may not exist. To add CA certificate info download  `cacert.pem` from [this](http://curl.haxx.se/docs/caextract.html) page and set `curl.cainfo` php ini variable to the location of `cacert.pem` e.g

{% codeblock php.ini lang:ini %}
curl.cainfo = /path/to/cacert.pem
{% endcodeblock %}

Then register application of the web service you want to use for authentication. For this post I have used Github for its simplicity. You can create application from [here](https://github.com/settings/applications/new). Your registration form may look like following,

{% img http://i.imgur.com/bfOhYma.png?2 %}

After successful application creation you will be redirected to application page where you will see `client ID` and `Client Secret` fields set for the application. They will be used later.

Add the bundle info in `composer.json` and issue `php composer.phar update --prefer-dist` command.

{% codeblock composer.json lang:javascript %}
{
    require: {
        //...
        "friendsofsymfony/user-bundle": "v1.3.2",
        "hwi/oauth-bundle": "0.3.*@dev",
    }
}
{% endcodeblock %}

Enable the bundles in `app/AppKernel.php`,

{% codeblock app/AppKernel.php lang:php  %}

public function registerBundles()
{
    $bundles = array(
        // ...
        new FOS\UserBundle\FOSUserBundle(),
        new HWI\Bundle\OAuthBundle\HWIOAuthBundle(),
    );
}

{% endcodeblock %}

Now setup `FOSUserBundle`. For this tutorial I will only show user entity creation and configuration. For other setup [refer to the documentation](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.md).


In one  of your bundle add entity class with field information. After that add a entity field named `githubID` which maps to the github user id. Minimal entity class is given bellow.

{% codeblock src/YourVendor/YourBundle/Entity/User.php lang:php %}
<?php

namespace YourVendor\YourBundle\Entity;

use FOS\UserBundle\Entity\User as BaseUser;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 * @ORM\Table(name="users")
 */
class User extends BaseUser
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @var string
     *
     * @ORM\Column(name="github_id", type="string", nullable=true)
     */
    private $githubID;


    public function __construct()
    {
        parent::__construct();
        // your own logic
    }

    /**
     * Get id
     *
     * @return integer 
     */
    public function getId()
    {
        return $this->id;
    }

}
{% endcodeblock %}



Add routes of `FOSUserBundle` in `app/config/rouging.yml`. Please note that I am securing parts of the site that matches with `^/secure_area` url pattern. So appropriate prefix was added in this case. To apply it in root url just remove `/secure_area` portion in all  occurrences.

{% codeblock app/config/routing.yml lang:php   %}
#...
fos_user_security:
    resource: "@FOSUserBundle/Resources/config/routing/security.xml"
    prefix: /secure_area

fos_user_profile:
    resource: "@FOSUserBundle/Resources/config/routing/profile.xml"
    prefix: /secure_area/profile

fos_user_register:
    resource: "@FOSUserBundle/Resources/config/routing/registration.xml"
    prefix: /secure_area/register

fos_user_resetting:
    resource: "@FOSUserBundle/Resources/config/routing/resetting.xml"
    prefix: /secure_area/resetting

fos_user_change_password:
    resource: "@FOSUserBundle/Resources/config/routing/change_password.xml"
    prefix: /secure_area/profile
{% endcodeblock %}


Add entity info in the `app/config/config.yml`

{% codeblock app/config/config.yml lang:ruby  %}
#...
fos_user:
    db_driver: orm # other valid values are 'mongodb', 'couchdb' and 'propel'
    firewall_name: secure_area
    user_class: YourVendor\YourBundle\Entity\User
    registration:
        confirmation:
            enabled:    false # change to true for required email confirmation

{% endcodeblock %}


Then in `app/config/security.yml` add `encoders` and `providers` information.

{% codeblock app/config/security.yml lang:ruby %}

security:
    encoders:
        FOS\UserBundle\Model\UserInterface: sha512
    providers:
        fos_userbundle:
            id: fos_user.user_provider.username

#...
{% endcodeblock %}

Now setup `HWIOauthBundle`. Add routes of `HWIOAuthBundle` to `app/config/routing.yml`.Another route named `hwi_github_login` was also added which is same as the callback url given during creation of Github application. This is the url which will be intercepted by the firewall to check authentication.

{% codeblock app/config/routing.yml lang:ruby %}

hwi_oauth_redirect:
    resource: "@HWIOAuthBundle/Resources/config/routing/redirect.xml"
    prefix:   /secure_area/connect

hwi_oauth_login:
    resource: "@HWIOAuthBundle/Resources/config/routing/login.xml"
    prefix:   /secure_area/connect

hwi_oauth_connect:
    resource: "@HWIOAuthBundle/Resources/config/routing/connect.xml"
    prefix:   /secure_area/connect

hwi_github_login:
    pattern: /secure_area/login/check-github

{% endcodeblock %}

Now setup the security firewall.

{% codeblock app/config/security.yml lang:ruby %}

security:
    #...
    firewalls:
        #...
        secure_area:
            pattern: ^/secure_area

            oauth:
                failure_path: /secure_area/connect
                login_path: /secure_area/connect
                check_path: /secure_area/connect
                provider: fos_userbundle
                resource_owners:
                    github:           "/secure_area/login/check-github"
                oauth_user_provider:
                    service: hwi_oauth.user.provider.fosub_bridge

            anonymous:    true
            logout:
                path:           /secure_area/logout
                target:         /secure_area/connect #where to go after logout

    #...

    access_control:
        #....
        - { path: ^/secure_area/login, roles: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/secure_area/connect, roles: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/secure_area, roles: ROLE_USER }

{% endcodeblock %}

In `firewalls` section a new firewall named `secure_area` with OAuth provider named `oauth` is added which handles `^/secure_area` url pattern. In `resource_owners` section of the OAuth provider intercept url for the Github resource owner is provided. It is same as the callback url given during Github application creation. 

In later `access_control` section path matching `^/secure_area/connect` and `^/secure_area/login` pattern moved out of secure area.

User provider of the OAuth authentication provider is `fos_userbundle` which was setup previously. As user provider is `FOSUserBundle`, built-in `hwi_oauth.user.provider.fosub_bridge` service was set as `oauth_user_provider`. If you want to set it to your custom user provider you have to implement [OAuthAwareUserProviderInterface](https://github.com/hwi/HWIOAuthBundle/blob/master/Security/Core/User/OAuthAwareUserProviderInterface.php).

Now setup `app/config/config.yml`.

{% codeblock app/config/config.yml lang:ruby   %}
#...
hwi_oauth:
    # name of the firewall in which this bundle is active, this setting MUST be set
    firewall_name: secure_area
    connect: 
        confirmation: true
        #account_connector: hwi_oauth.user.provider.fosub_bridge
        #registration_form_handler: hwi_oauth.registration.form.handler.fosub_bridge
        #registration_form: fos_user.registration.form

    resource_owners:
        github:
            type:                github
            client_id:           <client_id>
            client_secret:       <client_secret>
            scope:               "user:email"

    fosub:
        # try 30 times to check if a username is available (foo, foo1, foo2 etc)
        username_iterations: 30

        # mapping between resource owners (see below) and properties
        properties:
            github: githubID

{% endcodeblock %}

The value of `firewall_name` is same as the name of the firewall with OAuth provider setup in `app/config/security.yml`.

In `resource_owners` section OAuth information were added. The value of `client_id` and `client_secret` are the values set by Github after the creation of the application. For configuration of other resource owners [see the documentation](https://github.com/hwi/HWIOAuthBundle/tree/master/Resources/doc/resource_owners).

Since `FOSUserBundle` were used as user provider, `fosub` section were added. In `properties` section `githubID` entity field was set as value of `github` config field.

The `connect` section connects `HWIOAuthBundle` to the registration system of Symfony. It also links existing logged in users to the authenticated service. Note that simply adding `connect: ~` would be enough to link `HWIOAuthBundle` to the registration system. For the brief explanation of the options I have added default values.

If `confirmation` option is set to true, user will be shown a  page that will ask the user to connect the current authenticated resource to existing logged in user account. The template location is [HWIOAuthBundle:Connect:connect_confirm.html.twig](https://github.com/hwi/HWIOAuthBundle/blob/master/Resources/views/Connect/connect_confirm.html.twig). To override the template [see the documentation](http://symfony.com/doc/current/book/templating.html#overriding-bundle-templates).

The value of `account_connector` is a user provider class that implements [AccountConnectorInterFace](https://github.com/hwi/HWIOAuthBundle/blob/master/Connect/AccountConnectorInterface.php). By default it is set to same `hwi_oauth.user.provider.fosub_bridge` service that was set in OAuth firewall. So if you want to add support for your custom user provider you have to extend it so that it implements [AccountConnectorInterFace](https://github.com/hwi/HWIOAuthBundle/blob/master/Connect/AccountConnectorInterface.php) and [OAuthAwareUserProviderInterface](https://github.com/hwi/HWIOAuthBundle/blob/master/Security/Core/User/OAuthAwareUserProviderInterface.php).

The `registration_form_handler` is set to `hwi_oauth.registration.form.handler.fosub_bridge` service. It is used during registration process and does almost same thing as default `FOSUserBundle` registration form handler. The difference is that it implements [RegistrationFormHandlerInterface](https://github.com/hwi/HWIOAuthBundle/blob/master/Form/RegistrationFormHandlerInterface.php). So if you want to add your custom handler you have to extend the handler to implement `RegistrationFormHandlerInterface`.

The value of `registration_form` is same as default `FOSUserBundle` registration form `fos_user.registration.form`. It is used during registration operation. The twig template of the registration file is at [HWIOAuthBundle:Connect:registration.html.twig](https://github.com/hwi/HWIOAuthBundle/blob/master/Resources/views/Connect/registration.html.twig). Override it to meet your requirement.

Then issue following commands which will generate entity setter/getter methods and save table information to DB.

{% codeblock command lang:sh %}
 php app/console doctrine:generate:entities YourVendorYourBundle
 php app/console doctrine:schema:update --force
{% endcodeblock %}


Thats all. Now go to any url matcing `^/sceure_area` pattern and you will be redirected to `/secure_area/connect` url where lists of OAuth resource owners will be shown. The twig template of the page is [HWIOAuthBundle:Connect:login.html.twig](https://github.com/hwi/HWIOAuthBundle/blob/master/Resources/views/Connect/login.html.twig). Override it to meet your requirement. After successful OAuth authentication new user will be redirected to registration page or to previous page if the user already exists.

Once first resource owner is configured adding other resource owners is very easy. Just add mapping resource owners field in the entity, add check-resource route on `app/config/routng.yml`, add client id and client secret to `app/config/config.yml`, add property mapping and add another line in `resource_owners` section of the `app/config/security.yml`.

Another bonus tip, After successful authentication you can get access token of the resource from the toke of the `security.context` service as `HWIOAuthBundle` sets [OAuthToken](https://github.com/hwi/HWIOAuthBundle/blob/master/Security/Core/Authentication/Token/OAuthToken.php#L22) after successful authentication. So just by adding following line

{% codeblock YourController.php lang:php %}
$accessToken = $this->get('security.context')->getToken()->getAccessToken();
{% endcodeblock %}

will give you the access token with which you can do REST API call to the resource.

I have combined code example of this post and my [previous post](http://m2mdas.github.io/blog/2013/11/18/integrate-fosuserbundle-and-sonatauserbundle-easily/) and uploaded to [Github](https://github.com/m2mdas/MinimalSecurityBundlesSetup). It integrates `FOSUserBundle`, `SonataAdminBundle`, `SonataUserBundle` and `HWIOAuthBundle`. Enjoy.

