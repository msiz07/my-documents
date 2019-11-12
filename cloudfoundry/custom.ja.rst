.. vim:set fenc=utf-8
  original: https://github.com/cloudfoundry/docs-buildpacks/blob/63f2444d871b65d68ef321585f232b414d20eb1c/custom.html.md.erb
  committed on 20 Sep 2019

.. |CFAR| replace:: Cloud Foundry Application Runtime (CFAR)

::::::::::::::::::::::::::::::::::::::::::::::::::
Creating Custom Buildpacks
::::::::::::::::::::::::::::::::::::::::::::::::::

このトピックはCloud Foundry Application Runtime用のbuildpackをどのように作成するかを説明します。

buildpackがどのように動作するかについてのさらなる情報は、\ :doc:`How Buildpacks Work in Cloud Foundry <./understand-buildpacks.ja>`\ トピックを参照してください。


##################################################
Package Custom Buildpacks
##################################################
.. _packaging-custom-buildpacks:

|CFAR|\ のbuildpackは、限定されたインターネット接続もしくはインターネット接続なしで動作できます。
`buildpack-packager <https://github.com/cloudfoundry/libbuildpack/tree/master/packager>`_\ は、部分的にもしくは完全に切断された環境での動作を可能にする、\ |CFAR|\ のbuildpackと柔軟性を与えます。

==================================================
Use the Buildpack Packager
==================================================
.. _use-buildpack-packager:

1. `buildpack-packager <https://github.com/cloudfoundry/libbuildpack/tree/master/packager>`_\ がインストールされていることを確認します。
2. buildpack内にmanifest.ymlを作成します。
3. cachedモードでpackagerを実行します：

  .. code::

      $ buildpack-packager build -cached -any-stack

packagerはbuildpackディレクトリの（ほとんど）すべてをzipファイルに追加します。manifestで除外するよう示されているものはどれも除外されます。

cachedモードでは、packagerはmanifestに記述された依存対象をダウンロードし追加します。

さらなる情報は、\ `buildpack-packager GitHub repository <https://github.com/cloudfoundry/libbuildpack/tree/master/packager>`_\ を参照してください。

==================================================
Use and Share the Packaged Buildpack
==================================================
.. _share-buildpack-package:

After you have packaged your buildpack using ``buildpack-packager`` you can use the resulting ``.zip`` file locally, or share it with others by uploading it to any network location that is accessible to the CLI. Users can then specify the buildpack with the `-b` option when they push apps. See `Deploying Apps with a Custom Buildpack <deploying-with-custom-buildpacks>`_ for details.

.. Note::

  Offline buildpack packages may contain proprietary dependencies that require distribution licensing or export control measures. For more information about offline buildpacks, refer to `Packaging Dependencies for Offline Buildpacks <https://docs.cloudfoundry.org/buildpacks/depend-pkg-offline.html#offline-buildpacks>`_.

You can also use the cf create-buildpack command to upload the buildpack into your Cloud Foundry deployment, making it accessible without the ``-b`` flag:

.. code::

    $ cf create-buildpack BUILDPACK PATH POSITION

You can find more documentation in the `Managing Custom Buildpacks <https://docs.cloudfoundry.org/adminguide/buildpacks.html>`_ topic.

==================================================
Specify a Default Version
==================================================
.. _specifying-default-versions:

As of ``buildpack-packager`` `version 2.3.0 <https://github.com/cloudfoundry/buildpack-packager/releases/tag/v2.3.0>`_, you can specify the default version for a dependency by adding a ``default_versions`` object to the ``manifest.yml`` file.
The ``default_versions`` object has two properties, ``name`` and ``version``. For example:

.. code:: yaml

    default_versions:
    - name: go
      version: 1.6.3
    - name: other-dependency
      version: 1.1.1

To specify a default version:

1. Add the ``default_version`` object to your manifest, following the `rules <rules>`_ below.
   For a complete example, see `manifest.yml <https://github.com/cloudfoundry/go-buildpack/blob/master/manifest.yml>`_ in the go-buildpack repository in GitHub.

2. Run the ``default_version_for`` script from the `compile-extensions <https://github.com/cloudfoundry/compile-extensions>`_ repository, passing the path of your ``manifest.yml`` and the dependency name as arguments. The following command uses the example manifest from step 1:

  .. code::

      $ ./compile-extensions/bin/default_version_for manifest.yml go 1.6.3

--------------------------------------------------
Rules for Specifying a Default Version
--------------------------------------------------

The ``buildpack-packager`` script validates this object according to the following rules:

* You can create at most one entry under ``default_versions`` for a single dependency. The following example causes ``buildpack-packager`` to fail with an error because the manifest specifies two default versions for the same ``go`` dependency.

  .. code:: yaml

      # Incorrect; will fail to package
      default_versions:
      - name: go
        version: 1.6.3
      - name: go
        version: 1.7.5

* If you specify a ``default_version`` for a dependency, you must also list that dependency and version under the ``dependencies`` section of the manifest. The following example causes ``buildpack-packager`` to fail with an error because the manifest specifies ``version: 1.9.2`` for the ``go`` dependency, but lists ``version: 1.7.5`` under ``dependencies``.

  .. code:: yaml

    # Incorrect; will fail to package
    default_versions:
    - name: go
      version: 1.9.2

    dependencies:
    - name: go
      version: 1.7.5
      uri: https://storage.googleapis.com/golang/go1.7.5.linux-amd64.tar.gz
      md5: c8cb76e2308c792e2705c2eb1b55de95
      cf_stacks:
      - cflinuxfs3

##################################################
Core Buildpack Communication Contract
##################################################
.. _contract:

このセクションは\ |CFAR|\ のコアのbuildpackが従っている、連携するための約束事を説明します。開発者が複数のbuildpack (multiple buildpacks) をアプリケーションで使用できるようにするために、この約束事はbuildpackがお互いにやり取りすることを可能にします。

buildpack開発者は自分のカスタムbuildpackがこの約束事に従うことを保証する必要があります。

このセクションは、以下のplaceholderを使用します：

* ``IDX``\ は0でパディングされたインデックスであり、優先度リストの中でのbuildpackの位置に対応します。
* ``MD5``\ はビルドパックのURLのMD5チェックサムです。

``/bin/supply``\ をとおして依存対象を提供するすべてのbuildpackは：


* 後続のbuildpackへ名前を提供するために、buildpackは\ ``/tmp/deps/IDX/config.yml``\ を作成する必要があります。このファイルは、後続のbuildpackのために、その他の様々な設定を含んでいても構いません。
* ``config.yml``\ ファイルは以下のような形式にして、\ ``BUILDPACK``\ を依存対象を提供するbuildpackの名前に、\ ``YAML-OBJECT``\ をbuildpack固有の設定を含んだYAMLオブジェクトに置き換えたものにするべきです：

  .. code::

      name: BUILDPACK
      config: YAML-OBJECT

* 後続のbuildpackに依存対象を提供するために、/tmp/deps/IDXの中に、以下のディレクトリが作成されても構いません：

  * ``/bin``: Contains binaries intended for $PATH during staging and launch
  * ``/lib``: Contains libraries intended for $LD_LIBRARY_PATH during staging and launch
  * ``/include``: Contains header files intended for compilation during staging
  * ``/pkgconfig``: Contains ``pkgconfig`` files intended for compilation during staging
  * ``/env``: Contains environment vars intended for staging, loaded as ``FILENAME=FILECONTENTS``
  * ``/profile.d``: Contains scripts intended for ``/app/.profile.d``, sourced before launch

* The buildpack may make use of previous non-final buildpacks by scanning ``/tmp/deps/`` for index-named directories containing ``config.yml``.

最後のbuildpackは：

* 先行して適用されたbuildpackによって提供された依存対象を利用するために、最後のbuildpackは、\ ``config.yml``\ を含み名前がインデックス (IDX) であるディレクトリのために、 \ ``/tmp/deps``\ を調べる必要があります。
* 先行のbuildpackによって提供された依存対象を利用するために、最後のbuildpackは：

  * May use ``/bin`` during staging, or make it available in $PATH during launch
  * May use ``/lib`` during staging, or make it available in $LD\_LIBRARY\_PATH during launch
  * May use ``/include``, ``/pkgconfig``, or ``/env`` during staging
  * May copy files from ``/profile.d`` to ``/tmp/app/.profile.d`` during staging
  * May use the supplied config object in ``config.yml`` during the staging process

##################################################
Deploy Apps with a Custom Buildpack
##################################################
.. _deploying-with-custom-buildpacks:

Once a custom buildpack has been created and pushed to a public git repository, the git URL can be passed via the cf CLI when pushing an app.

For example, for a buildpack that has been pushed to GitHub:

.. code::

    $ cf push my-new-app -b git://github.com/johndoe/my-buildpack.git

Alternatively, you can use a private git repository, with https and username/password authentication, as follows:

.. code::

    $ cf push my-new-app -b https://username:password@github.com/johndoe/my-buildpack.git

By default, |CFAR| uses the default branch of the buildpack's git repository. You can specify a different branch using the git url as shown in the following example:

.. code::

    $ cf push my-new-app -b https://github.com/johndoe/my-buildpack.git#my-branch-name

Additionally, you can use tags in a git repository, as follows:

.. code::

    $ cf push my-new-app -b https://github.com/johndoe/my-buildpack#v1.4.2

The app will then be deployed to |CFAR|, and the buildpack will be cloned from the repository and applied to the app.

.. Note:: If a buildpack is specified using ``cf push -b`` the ``detect`` step will be skipped and as a result, no buildpack ``detect`` scripts will be run.

##################################################
Disabling Custom Buildpacks
##################################################

Operators can choose to disable custom buildpacks. For more information, see `Disabling Custom Buildpacks <https://docs.cloudfoundry.org/adminguide/buildpacks.html#disabling-custom-buildpacks>`_.

.. Note:: A common development practice for custom buildpacks is to fork existing buildpacks and sync subsequent patches from upstream. To merge upstream patches to your custom buildpack, use the approach that GitHub recommends for `syncing a fork <https://help.github.com/articles/syncing-a-fork>`_.

