---
layout: post
title: "Integrate FOSUserBundle and SonataUserBundle easily"
date: 2013-11-18 05:41
comments: true
categories: [PHP, Symfony, FOSUserBundle, SonataUserBundle, SonataAdminBundle]
description: Easy way to integrate FOSUserBundle with SonataUserBundle Symfony2 bundles
keywords: PHP, Symfony2, FOSUserBundle, SonataUserBundle, FOSUserBundle + SonataUserBundle, integrate FOSUserBundle with SonataUserBundle, SonataAdminBundle, SonataUserBundle, Setup, integrating, configuration, install, Easy
---
[SonataUserBundle](https://github.com/sonata-project/SonataUserBundle) is a great extension of [SonataAdminBundle](https://github.com/sonata-project/SonataAdminBundle) that provides user administration features by integrating [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle) user provider/management bundle. Its [default installation procedure](https://github.com/sonata-project/SonataUserBundle/blob/master/Resources/doc/reference/installation.rst#installation) recommends to setup `SonataUserBundle` as child bundle of `FOSUserBundle` and generate `ApplicationSonataUserBundle` via `sonata:easy-extends:generate` command. But on some cases you may not want to setup that way. For example you have setup your user entity by following the documentation of `FOSUserBundle` before integrating `SonataAdminBundle` and `SonataUserBundle`, you may want to override both bundles separately.  In following section I will outline how to integrate `SonataUserBundle` with `FOSUserBundle` without creating child bundle of `FOSUserBundle`.

Default installation and setup of `FOSUserBundle` is same as [stated in default documentation](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.md#getting-started-with-fosuserbundle). Then install and setup `SonataAdminBundle` [according to its documentation](http://sonata-project.org/bundles/admin/master/doc/index.html#reference-guide). Both guides are pretty well documented. So I will not duplicate them here.

Now setup `SonataUserBundle`. Add it to `composer.json` and add following line into `registerBundle` method of `app/AppKernel.php`

{% codeblock app/AppKernel.php lang:php %}
// app/AppKernel.php
public function registerBundles()
{

    $bundles = array(
        // other bundle declarations
        new Sonata\UserBundle\SonataUserBundle(),
    );
}
{% endcodeblock %}

Then setup configuration, add routing and security configuration according to [the documentation](https://github.com/sonata-project/SonataUserBundle/blob/master/Resources/doc/reference/installation.rst#configuration).

Now set value of `sonata.user.admin.user.class` parameter to the FQCN of the User entity which was created during `FOSUserBundle` setup. For example if FQCN of your user entity is `YourVendor\YourBundle\Entity\User` then parameter setting of `app/config.yml` would be 

{% codeblock app/config/config.yml lang:ruby %}
parameters:
    #....
    sonata.user.admin.user.entity: YourVendor\YourBundle\Entity\User
{% endcodeblock %}

Now create a class that extends default [UserAdmin](https://github.com/sonata-project/SonataUserBundle/blob/master/Admin/Model/UserAdmin.php) class and override `configureShowFields`, `configureFormFields`, `configureDatagridFilters` and `configureListFields` methods to add the needed user admin fields. Following is the sample extended `UserAdmin` class which is based on the bare bone user entity created in `FOSUserBundle` documentation.

{% codeblock src/YourVendor/YourBundle/Admin/UserAdmin.php lang:php  %}

<?php
//src/YourVendor/YourBundle/Admin/UserAdmin.php

namespace YourVendor\YourBundle\Admin;

use Sonata\UserBundle\Admin\Model\UserAdmin as BaseUserAdmin;

use Sonata\AdminBundle\Form\FormMapper;
use Sonata\AdminBundle\Datagrid\DatagridMapper;
use Sonata\AdminBundle\Datagrid\ListMapper;
use Sonata\AdminBundle\Show\ShowMapper;

use FOS\UserBundle\Model\UserManagerInterface;
use Sonata\AdminBundle\Route\RouteCollection;


class UserAdmin extends BaseUserAdmin
{
    /**
     * {@inheritdoc}
     */
    protected function configureShowFields(ShowMapper $showMapper)
    {
        $showMapper
            ->with('General')
                ->add('username')
                ->add('email')
            ->end()
            // .. more info
        ;
    }

    /**
     * {@inheritdoc}
     */
    protected function configureFormFields(FormMapper $formMapper)
    {

        $formMapper
            ->with('General')
                ->add('username')
                ->add('email')
                ->add('plainPassword', 'text', array('required' => false))
            ->end()
            // .. more info
            ;

        if (!$this->getSubject()->hasRole('ROLE_SUPER_ADMIN')) {
            $formMapper
                ->with('Management')
                    ->add('roles', 'sonata_security_roles', array(
                        'expanded' => true,
                        'multiple' => true,
                        'required' => false
                    ))
                    ->add('locked', null, array('required' => false))
                    ->add('expired', null, array('required' => false))
                    ->add('enabled', null, array('required' => false))
                    ->add('credentialsExpired', null, array('required' => false))
                ->end()
            ;
        }
    }

    /**
     * {@inheritdoc}
     */
    protected function configureDatagridFilters(DatagridMapper $filterMapper)
    {
        $filterMapper
            ->add('id')
            ->add('username')
            ->add('locked')
            ->add('email')
        ;
    }
    /**
     * {@inheritdoc}
     */
    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->addIdentifier('username')
            ->add('email')
            ->add('enabled', null, array('editable' => true))
            ->add('locked', null, array('editable' => true))
            ->add('createdAt')
        ;

        if ($this->isGranted('ROLE_ALLOWED_TO_SWITCH')) {
            $listMapper
                ->add('impersonating', 'string', array('template' => 'SonataUserBundle:Admin:Field/impersonating.html.twig'))
            ;
        }
    }
}
{% endcodeblock %}

Now set the value of `sonata.user.admin.user.class` to the FQCN of the created `UserAdmin` class in `app/config/config.yml`, e.g

{% codeblock app/config/config.yml lang:ruby %}

#app/config/config.yml

parameters:
    # ...
    sonata.user.admin.user.class: YourVendor\YourBundle\Admin\UserAdmin

# ....
{% endcodeblock %}

If you don't need user group functionality you can disable it. e.g in your `app/config/config.yml` add following lines.

{% codeblock app/config/config.yml lang:ruby %}

#app/config/config.yml

services:
    sonata.user.admin.group:
        abstract: true
        public: false

{% endcodeblock %}

Combined `config.yml` setting given bellow,

{% codeblock app/config/config.yml lang:ruby %}

#app/config/config.yml

#...

parameters:
    sonata.user.admin.user.class: YourVendor\YourBundle\Admin\UserAdmin
    sonata.user.admin.user.entity: YourVendor\YourBundle\Entity\User

services:
    sonata.user.admin.group:
        abstract: true
        public: false
#...

{% endcodeblock %}

If everything setup correctly you will see an new users row in `admin/dashboard` page. All user operations should work as expected. Thats all for now.
