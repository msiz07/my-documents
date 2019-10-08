
..
  commit: bb91213
  Commits on My 23, 2019

..
  title: How Buildpacks Work in Cloud Foundry
  owner: Buildpacks

..
  This topic describes how buildpacks are structured and detected in Cloud Foundry.

.. ::

  ## <a id='buildpack-scripts'></a>Buildpack Scripts ##

  A buildpack repository may contain the following five scripts in the `bin` directory:

  * `bin/detect` determines whether or not to apply the buildpack to an app.
  * `bin/supply` provides dependencies for an app.
  * `bin/finalize` prepares the app for launch.
  * `bin/release` provides feedback metadata to Cloud Foundry indicating how the app should be executed.
  * `bin/compile` is a deprecated alternative to `bin/supply` and `bin/finalize`.

  The `bin/supply` and `bin/finalize` scripts replace the deprecated `bin/compile` script.
  Older buildpacks may still use `bin/compile` with the latest version of Cloud Foundry. In this case, applying multiple buildpacks to a single app is not supported.
  Similarly, newer buildpacks may still provide `bin/compile` for compatibility with Heroku and older versions of Cloud Foundry.

  The `bin/supply` script is required for non-final buildpacks. The `bin/finalize` (or `bin/compile`) script is required for final buildpacks.

  <p class="note"><strong>Note</strong>: In this document, the terms <em>non-final buildpack</em> and <em>final buildpack</em>, or <em>last buildpack</em>, are used to describe the process of applying multiple buildpacks to an app. See the following example: <code>cf push APP-NAME -b FIRST-BUILDPACK -b SECOND-BUILDPACK -b FINAL-BUILDPACK</code>.</p>

  <p class="note"><strong>Note</strong>: If you use only one buildpack for your app, this buildpack behaves as a final, or last, buildpack.</p>

  <p class="note"><strong>Note</strong>: When using multi-buildpack support, the last buildpack in order is the final buildpack, and is able to make changes to the app and determine a start command. All other specified buildpacks are non-final and only supply dependencies.</p>


.. ::

  ### <a id='detect-script'></a>bin/detect ###

  The `detect` script determines whether or not to apply the buildpack to an app. The script is called with one argument, the `build` directory for the app. The `build` directory contains the app files uploaded when a user performs a `cf push`. 

  The `detect` script returns an exit code of `0` if the buildpack is compatible with the app. In the case of system buildpacks, the script also prints the buildpack name, version, and other information to `STDOUT`.

  The following is an example `detect` script that checks for a Ruby app based on the existence of a `Gemfile`:

  ~~ruby
  #!/usr/bin/env ruby

  gemfile_path = File.join ARGV[0], "Gemfile"

  if File.exist?(gemfile_path)
    puts "Ruby"
    exit 0
  else
    exit 1
  end
  ~~

  Optionally, the buildpack `detect` script can output additional details provided by the buildpack developer. This includes buildpack versioning information and a list of configured frameworks and their associated versions.

  The following is an example of the detailed information returned by the Java buildpack:

  ~~
  java-buildpack=v3.0-https://github.com/cloudfoundry/java-buildpack.git#3bd15e1 open-jdk-jre=1.8.0_45 spring-auto-reconfiguration=1.7.0_RELEASE tomcat-access-logging-support=2.4.0_RELEASE tomcat-instance=8.0.21 tomcat-lifecycle-support=2.4.0_RELEASE ...
  ~~

  <p class="note"><strong>Note</strong>: Cloud Foundry detects only one buildpack by default. When multiple buildpacks are desired, you must explicitly specify them.</p>

  For more information, see the [Buildpack Detection](#buildpack-detection) section below.


.. ::

  ### <a id='supply-script'></a>bin/supply ###

  The `supply` script provides dependencies for the app and runs for all buildpacks. All output sent to `STDOUT` is relayed to the user through the Cloud Foundry Command Line Interface (cf CLI).

  The script is run with four arguments:

  * The `build` directory for the app
  * The `cache` directory, which is a location the buildpack can use to store assets during the build process
  * The `deps` directory, which is where dependencies provided by all buildpacks are installed
  * The `index`, which is a number that represents the ordinal position of the buildpack

  The `supply` script stores dependencies in `deps`/`index`. It may also look in other directories within `deps` to find dependencies supplied by other buildpacks.

  The `supply` script must not modify anything outside of the `deps`/`index` directory. Staging may fail if such modification is detected.

  The `cache` directory provided to the `supply` script of the final buildpack is preserved even when the buildpack is upgraded or otherwise changes. The `finalize` script also has access to this cache directory.

  The `cache` directories provided to the `supply` scripts of non-final buildpacks are cleared if those buildpacks are upgraded or otherwise change.

  The following is an example of a simple `supply` script:


  ~~ruby
  #!/usr/bin/env ruby

  #sync output

  $stdout.sync = true

  build_path = ARGV[0]
  cache_path = ARGV[1]
  deps_path = ARGV[2]
  index = ARGV[3]

  install_ruby

  private

  def install_ruby
    puts "Installing Ruby"

    # !!! build tasks go here !!!
    # download ruby 
    # install ruby
    end
  ~~

.. ::

  ### <a id='finalize-script'></a>bin/finalize ###

  The `finalize` script prepares the app for launch and runs only for the last buildpack. All output sent to `STDOUT` is relayed to the user through the cf CLI.

  The script is run with four arguments:

  * The `build` directory for the app
  * The `cache` directory, which is a location the buildpack can use to store assets during the build process
  * The `deps` directory, which is where dependencies provided by all buildpacks are installed
  * The `index`, which is a number that represents the ordinal position of the buildpack

  The `finalize` script may find dependencies installed by the `supply` script of the same buildpack in `deps`/`index`. It may also look in other directories within `deps` to find dependencies supplied by other buildpacks.

  The `cache` directory provided to the `finalize` script is preserved even when the buildpack is upgraded or otherwise changes. The `supply` script of the same buildpack also has access to this cache directory.

  The following is an example of a simple `finalize` script:

  ~~ruby
  #!/usr/bin/env ruby

  #sync output

  $stdout.sync = true

  build_path = ARGV[0]
  cache_path = ARGV[1]
  deps_path = ARGV[2]
  index = ARGV[3]

  setup_ruby

  private

  def setup_ruby
    puts "Configuring your app to use Ruby"

    # !!! build tasks go here !!!
    # setup ruby 
  end
  ~~

.. ::

  ### <a id='compile-script'></a>bin/compile (Deprecated) ###

  The `compile` script is deprecated. It encompasses the behavior of the `supply` and `finalize` scripts for single buildpack apps by using the `build` directory to store dependencies.

  The script is run with two arguments:

  * The `build` directory for the app
  * The `cache` directory, which is a location the buildpack can use to store assets during the build process

  During the execution of the `compile` script, all output sent to `STDOUT` is relayed to the user through the cf CLI.

.. ::

  ### <a id='release-script'></a>bin/release ###

  The `release` script provides feedback metadata to Cloud Foundry indicating how the app should be executed. The script is run with one argument, the `build` directory. The script must generate a YAML file in the following format:


  ~~yaml
  default_process_types:
    web: start_command.filetype
  ~~

  `default_process_types` indicates the type of app being run and the command used to start it. 
  This start command is used if a start command is not specified in the `cf push` or in a Procfile.

  At this time, only the `web` type of apps is supported.

  <p class="note"><strong>Note</strong>: To define environment variables for your buildpack, add a Bash script to the <code>.profile.d</code> directory in the root folder of your app.</p>

  The following example shows what a `release` script for a Rack app might return:


  ~~ruby
  default_process_types:
    web: bundle exec rackup config.ru -p $PORT
  ~~

  <p class="note"><strong>Note</strong>: The <code>web</code> command runs as <code>bash -c COMMAND</code> when Cloud Foundry starts your app. Refer to <a href="../devguide/deploy-apps/manifest.html#start-commands">the command attribute</a> section for more information about custom start commands. </p>

.. ::

  ## <a id='droplet-filesystem'></a> Droplet Filesystem ##

  The buildpack staging process extracts the droplet into the `/home/vcap` directory inside the instance container and creates the following filesystem tree:

  ```
  app/
  deps/
  logs/
  tmp/
  staging_info.yml
  ```

  The `app` directory includes the contents of the `build` directory, and `staging_info.yml` contains the staging metadata saved in the droplet.

.. ::

  ##<a id='buildpack-detection'></a> Buildpack Detection

  When you push an app, Cloud Foundry uses a detection process to determine a single buildpack to use. 

  For general information about this process, see [How Apps Are Staged](../concepts/how-applications-are-staged.html#stage-buildpack). 

  During staging, each buildpack has a position in a priority list. You can retrieve this position by running `cf buildpacks`.

  Cloud Foundry checks if the buildpack in position 1 is a compatible buildpack. If the position 1 buildpack is not compatible, Cloud Foundry moves on to the buildpack in position 2. Cloud Foundry continues this process until the correct buildpack is found. 

  If no buildpack is compatible, the `cf push` command fails with the following error:

  <pre class="terminal">
  None of the buildpacks detected a compatible application
  Exit status 222
  Staging failed: Exited with status 222

  FAILED
  NoAppDetectedError
  </pre>

  For a more detailed account of how Cloud Foundry interacts with the buildpack, see the [Sequence of Interactions](#interactions) section below.

.. ::

  ##<a id='interactions'></a> Sequence of Interactions 

  This section describes the sequence of interactions between the Cloud Foundry platform and the buildpack. The sequence of interactions differs depending on whether the platform [skips](#no-detection) or [performs](#detection) buildpack detection.

.. ::

  ###<a id='no-detection'></a> No Buildpack Detection

  Cloud Foundry skips buildpack detection if the developer specifies one or more buildpacks in the app manifest or in the `cf push APP-NAME -b BUILDPACK-NAME` cf CLI command.

  If you explicitly specify buildpacks, Cloud Foundry performs the following interactions:

  1. For each buildpack except the last buildpack, the platform does the following:
    1. Creates the `deps`/`index` directory
    1. Runs `/bin/supply` with the `build`, `cache`, and `deps` directories and the buildpack `index`
    1. Accepts any modification of the `deps`/`index` directory
    1. Accepts any modification of the `cache` directory
    1. May disallow modification of any other directories
  1. For the last buildpack, the platform does the following:
    1. If `/bin/finalize` is present:
      1. Creates the `deps`/`index` directory if it does not exist
      1. If `/bin/supply` is present, runs `/bin/supply` with the `build`, `cache`, and `deps` directories and the buildpack `index`
      1. Accepts any modification of the `deps`/`index` directory
      1. May disallow modification of the `build` directory
      1. Runs `/bin/finalize` with the `build`, `cache`, and `deps` directories and the buildpack `index`
      1. Accepts any modification of the `build` directory
    1. If `/bin/finalize` is not present:
      1. Runs `/bin/compile` with the `build` and `cache` directories
      1. Accepts any modification of the `build` directory
    1. Runs `/bin/release` to determine staging information

  At the end of this process, the `deps` directory is included at the root of the droplet, adjacent to the `app` directory.

.. ::

  ###<a id='detection'></a> Buildpack Detection

  Cloud Foundry performs buildpack detection if the developer does not specify one or more buildpacks in the app manifest or in the `cf push APP-NAME -b BUILDPACK-NAME` cf CLI command.

  <p class="note"><strong>Note</strong>: Cloud Foundry detects only one buildpack to use with the app.</p>

  If the platform performs detection, it does the following:

  1. Runs `/bin/detect` for each buildpack
  1. Selects the first buildpack with a `/bin/detect` script that returns a zero exit status
  1. If `/bin/finalize` is present:
    1. Creates the `deps`/`index` directory if it does not exist
    1. If `/bin/supply` is present, runs `/bin/supply` with the `build`, `cache`, and `deps` directories and the buildpack `index`
    1. Accepts any modification of the `deps`/`index` directory
    1. May disallow modification of the `build` directory
    1. Runs `/bin/finalize` on the `build`, `cache`, and `deps` directories
    1. Accepts any modification of the `build` directory
  1. If `/bin/finalize` is not present:
    1. Runs `/bin/compile` on the `build` and `cache` directories
    1. Accepts any modification of the `build` directory
  1. Runs `/bin/release` to determine staging information

  At the end of this process, the `deps` directory is included at the root of the droplet, adjacent to the `app` directory.

..
  --------------------------------------------------

::::::::::::::::::::::::::::::::::::::::::::::::::
How Buildpacks Work in Cloud Foundry
::::::::::::::::::::::::::::::::::::::::::::::::::

このトピックは、buildpackがどのような構造であり、Cloud Foundryの中でどのように探されるかを説明します。

##################################################
Buildpack Scripts
##################################################


buildpackのレポジトリには、以下の5つのスクリプトを\ ``bin``\ ディレクトリに含めることができます。

* ``bin/detect``\ は、buildpackをappに適用するかどうかを決定します。
* ``bin/supply``\ は、appの依存対象を提供します。
* ``bin/finalize``\ は、app起動の準備をします。
* ``bin/release``\ は、Cloud Foundryにfeedback metadataを提供し、appをどのように実行すべきかを示します。
* ``bin/compile``\ は、\ ``bin/supply``\ と\ ``bin/finalize``\ に対する廃棄予定の代替手段です。

``bin/supply``\ と\ ``bin/finalize``\ スクリプトは、廃棄予定の\ ``bin/compile``\ スクリプトを置き換えます。\
以前のbuildpackは、まだ\ ``bin/compile``\ を最新バージョンのCloud Foundryでも使用しているかもしれません。その場合、1つのappに複数のbuildpackを適用する機能はサポートされません。\
同様に、新しいbuildpackは、Herokuや古いバージョンのCloud Foundryとの互換性のために、\ ``bin/compile``\ をまだ提供しているかもしれません。

``bin/supply``\ スクリプトは、non-final buildpackでは必要とされます。\ ``bin/finalize``\ (または\ ``bin/compile``\ )スクリプトは、final buildpackでは必要とされます。

.. Note::

  この文書では、\ **non-final buildpack**\ および\ **final buildpack**\ または\ **last buildpack**\ という用語を、appに複数のbuildpackを適用するプロセスを説明するために使用します。以下の例を参照ください::

    cf push APP-NAME -b FIRST-BUILDPACK -b SECOND-BUILDPACK -b FINAL-BUILDPACK

.. Note::

  もしappにbuildpackを1つだけ使用するときは、そのbuildpackはfinal buildpackまたはlast buildpackとして振る舞います。

.. Note::

   複数buildpackのサポートを使用するときは、順番が最後のbuildpackが、final buildpackになります。final buildpackは、appの変更や開始コマンドの決定ができます。(複数buildpackサポートで)指定されたその他の全てのbuildpackはnon-final buildpackとなり、依存対象の提供だけを行います。

==================================================
bin/detect
==================================================

``detect``\ スクリプトは、buildpackをappへ適用するかどうかを決定します。このスクリプトは、appの\ ``build``\ ディレクトリを示す1つの引数と一緒に呼び出されます。\ ``build``\ ディレクトリは、ユーザが\ ``cf push``\ を実施したときにアップロードされるappのファイルを収容します。

``detect``\ スクリプトは、buildpackがappと両立できるときには、終了コード\ ``0``\ を返します。システムbuildpackの場合は、さらにbuildpackの名称、バージョン、その他の情報を\ ``STDOUT``\ へ表示します。

以下は、\ ``Gemfile``\ の存在に基づいてRubyのappをチェックする\ ``detect``\ スクリプトの例です。

.. code-block:: ruby

  #!/usr/bin/env ruby

  gemfile_path = File.join ARGV[0], "Gemfile"

  if File.exist?(gemfile_path)
    puts "Ruby"
    exit 0
  else
    exit 1
  end

適宜、buildpackの\ ``detect``\ スクリプトは、buildpack開発者によって提供される追加の詳細情報を出力できます。これはbuildpackのバージョン情報および構成されたフレームワークと関連するバージョンのリストを含みます。

以下は、Java buildpackによって返される詳細情報の例です。

::

  java-buildpack=v3.0-https://github.com/cloudfoundry/java-buildpack.git#3bd15e1 open-jdk-jre=1.8.0_45 spring-auto-reconfiguration=1.7.0_RELEASE tomcat-access-logging-support=2.4.0_RELEASE tomcat-instance=8.0.21 tomcat-lifecycle-support=2.4.0_RELEASE ...

.. Note::

   Cloud Foundryは、初期設定ではbuildpackを1つだけ検出します。複数のbuildpackを希望するときは、それらを明示的に指定する必要があります。

さらなる情報は、\ `Buildpack Detection <buildpack-detection>`_\ セクションを参照ください。


==================================================
bin/supply
==================================================

``supply``\ スクリプトはappの依存対象を提供し、全てのbuildpackで実行されます。\ ``STDOUT``\ へ送られる全ての出力は、Cloud Foundry Command Line Interface (cf CLI)をとおしてユーザへ中継されます。

このスクリプトは4つの引数と一緒に実行されます:

* app用の\ ``build``\ ディレクトリ
* ``cache``\ ディレクトリ、buildpackがビルドのプロセス中に資産(assets)を格納するために使用できる場所
* ``deps``\ ディレクトリ、全てのbuildpackによって提供される依存対象がインストールされる場所
* ``index``\ 、buildpackの順番上の位置を示す数字

``supply``\ スクリプトは、依存対象を\ ``deps``/``index``\ に格納します。他のbuildpackによって提供された依存対象を探すために、\ ``deps``\ 以下の他のディレクトリも調べるかもしれません。

``supply``\ スクリプトは、\ ``deps``/``index``\ ディレクトリの外側は何も変更してはいけません。そのような変更が検知された場合は、ステージング処理(訳注：buildpackに基づくビルド・デプロイなどの処理)は失敗するかもしれません。

final buildpackの\ ``supply``\ スクリプトへ提供された\ ``cache``\ ディレクトリは、そのbuildpackが更新または何かしらの変更がされたときも保持されます。\ ``finalize``\ スクリプトもcacheディレクトリへ自由にアクセスできます。

no-final buildpackの\ ``supply``\ スクリプトへ提供された\ ``cache``\ ディレクトリは、それらのbuildpackが更新または何かしらの変更がされたときは、クリアされます。

以下は、単純な\ ``supply``\ スクリプトの例です。

.. code-block:: ruby

  #!/usr/bin/env ruby

  #sync output

  $stdout.sync = true

  build_path = ARGV[0]
  cache_path = ARGV[1]
  deps_path = ARGV[2]
  index = ARGV[3]

  install_ruby

  private

  def install_ruby
    puts "Installing Ruby"

    # !!! build tasks go here !!!
    # download ruby 
    # install ruby
    end


==================================================
bin/finalize
==================================================

``finalize``\ スクリプトは、appの起動を準備し、last buildpackでだけ実行されます。\ ``STDOUT``\ へ送られる全ての出力は、cf CLIをとおしてユーザへ中継されます。

このスクリプトは4つの引数と一緒に実行されます。

* app用の\ ``build``\ ディレクトリ
* ``cache``\ ディレクトリ、buildpackがビルドのプロセス中に資産(assets)を格納するために使用できる場所
* ``deps``\ ディレクトリ、全てのbuildpackによって提供される依存対象がインストールされる場所
* ``index``\ 、buildpackの順番上の位置を示す数字

``finalize``\ スクリプトは、同じbuildpackの\ ``supply``\ スクリプトによって\ ``deps``/``index``\ にインストールされた依存対象を探しても構いません。他のbuildpackによって提供された依存対象を探すために、\ ``deps``\ 以下の他のディレクトリも調べるかもしれません。

``finalize``\ スクリプトへ提供された\ ``cache``\ ディレクトリは、そのbuildpackが更新または何かしらの変更がされたときも保持されます。同じbuildpackの\ ``supply``\ スクリプトもcacheディレクトリへ自由にアクセスできます。

以下は、単純な\ ``finalize``\ スクリプトの例です:

.. code-block:: ruby

  #!/usr/bin/env ruby

  #sync output

  $stdout.sync = true

  build_path = ARGV[0]
  cache_path = ARGV[1]
  deps_path = ARGV[2]
  index = ARGV[3]

  setup_ruby

  private

  def setup_ruby
    puts "Configuring your app to use Ruby"

    # !!! build tasks go here !!!
    # setup ruby 
  end


==================================================
bin/compile
==================================================

``compile``\ スクリプトは廃止予定です。これは、依存対象の格納に\ ``build``\ ディレクトリを使用して、buildpackが1つのappでの\ ``supply``\ と\ ``finalize``\ スクリプトの振る舞いを包含します。

このスクリプトは2つの引数と一緒に実行されます。

* app用の\ ``build``\ ディレクトリ
* ``cache``\ ディレクトリ、buildpackがビルドのプロセス中に資産(assets)を格納するために使用できる場所

``compile``\ スクリプトを実行している間、\ ``STDOUT``\ へ送られる全ての出力はcf CLIをとおしてユーザへ中継されます。


==================================================
bin/release
==================================================

``release``\ スクリプトは、Cloud Foundryにfeedback metadataを提供し、appをどのように実行すべきかを示します。このスクリプトは1つの引数、\ ``build``\ ディレクトリ、と一緒に実行されます。このスクリプトは、以下の形式のYAMLファイルを生成する必要があります:


.. code-block:: yaml

  default_process_types:
    web: start_command.filetype

``default_process_types``\ は、実行されようとしているappのタイプと、それを開始するときに使用するコマンドを示します。その開始コマンドは、\ ``cf push``\ の中か、またはProcfileの中で指定されなかった場合に、使用されます。

現時点では、\ ``web``\ タイプのappだけがサポートされています。

.. Note::

   buildpack用に環境変数を定義するときは、Bashスクリプトをappのルートフォルダ中の\ ``.profile.d``\ ディレクトリへ追加します。

以下は、Rack appの\ ``release``\ スクリプトが返しそうなものを示す例です:

.. code-block:: ruby

  default_process_types:
    web: bundle exec rackup config.ru -p $PORT

.. Note::

   ``web``\ コマンドは、Cloud Foundryがappを開始するときに、\ bash -c COMMAND</code>``\ として実行されます。開始コマンドのカスタマイズについてより詳細な情報は、(manifestの)\ `the command attribute <https://docs.cloudfoundry.org/devguide/deploy-apps/manifest.html#start-commands>`_\ セクションを参照してください。


##################################################
Droplet Filesystem
##################################################

buildpackのステージング・プロセスは、インスタンスのコンテナ内の\ ``/home/vcap``\ ディレクトリの中へdropletを展開し、以下のファイルシステム・ツリーを作成します:

.. code::

  app/
  deps/
  logs/
  tmp/
  staging_info.yml

``app``\ ディレクトリは\ ``build``\ ディレクトリのコンテンツを含み、\ ``staging_info.yml``\ はdroplet内に保存されているstaging metadataを含みます。


##################################################
Buildpack Detection
##################################################
.. _buildpack-detection:

appをpushするとき、使用するbuildpackを1つに決定するために、Cloud Foundryはdetectionプロセスを使用します。

このプロセスの概要については、\ `How Apps Are Staged <https://docs.cloudfoundry.org/concepts/how-applications-are-staged.html#stage-buildpack>`_\ を参照ください。

ステージングの間、各buildpackには優先度リスト中の位置が付けられます。\ ``cf buildpacks``\ を実行する
と、その位置を表示できます。

Cloud Foundryは、最初に位置するbuildpackが(appと)両立するbuildpackかどうかをチェックします。もし最初のbuildpackが両立しない場合は、Cloud Foundryは2番目のbuildpackへ進みます。Cloud Foundryは、両立するbuildpackが見つかるまで、このプロセスを継続します。

適用可能なbuildpackがなかった場合は、\ ``cf push``\ コマンドは以下のエラーを伴いながら失敗します:

.. code::

  None of the buildpacks detected a compatible application
  Exit status 222
  Staging failed: Exited with status 222

  FAILED
  NoAppDetectedError

Cloud Foundryがbuildpacktどのようにやり取りするかのより詳細な説明は、以下の\ `Sequence of Interactions`_\ セクションを参照ください。


##################################################
Sequence of Interactions
##################################################

このセクションは、Cloud Foundryプラットフォームとbuildpacktの間の一連のやり取りを説明します。この一連のやり取りは、プラットフォームがbuildpack detectedを\ `スキップ <no-detection_>`_\ するか\ `実行 <detection_>`_\ するかによって異なります。

==================================================
No Buildpack Detection
==================================================
.. _no-detection:

開発者が、app manifestの中もしくは\ `cf push APP-NAME -b BUILDPACK-NAME`\ cf CLIコマンドの中で、1つ以上のbuildpackを指定した場合、Cloud Foundryはbuildpack detectedをスキップします。

もしbuildpackを明示的に指定した場合、Cloud Foundryは以下のやり取りを実行します:

1. last buildpack以外の各buildpackで、プラットフォームは以下を実行します:

  #. ``deps``/``index``\ ディレクトリを作成します
  #. ``build``\ 、\ ``cache``\ 、\ ``deps``\ ディレクトリおよびbuildpackの\ ``index``\ と一緒に\ ``/bin/supply``\ を実行します
  #. ``deps``/``index``\ ディレクトリへの変更を受け入れます
  #. ``cache``\ ディレクトリへの変更を受け入れます
  #. その他のディレクトリへのあらゆる変更は許可されない場合があります

2.last buildpackで、プラットフォームは以下を実行します:

  1. もし\ ``/bin/finalize``\ が存在する場合:

    #. ``deps``/``index``\ ディレクトリが存在しない場合は、作成します
    #. ``/bin/supply``\ が存在する場合は、\ ``build``\ 、\ ``cache``\ 、\ ``deps``\ ディレクトリおよびbuildpackの\ ``index``\ と一緒に、\ ``/bin/supply``\ を実行します
    #. ``deps``/``index``\ ディレクトリへの変更を受け入れます
    #. ``build``\ ディレクトリへのあらゆる変更は許可されない場合があります
    #. ``build``\ 、\ ``cache``\ 、\ ``deps``\ ディレクトリおよびbuildpackの\ ``index``\ と一緒に、\ ``/bin/finalize``\ を実行します
    #. ``build``\ ディレクトリへの変更を受け入れます

  2. もし\ ``/bin/finalize``\ が存在しない場合:

    #. ``build``\ および\ ``cache``\ ディレクトリと一緒に\ ``/bin/compile``\ を実行します
    #. ``build``\ ディレクトリへの変更を受け入れます

  3. staging informationを決定するために\ ``/bin/release``\ を実行します

このプロセスの最後に、\ ``deps``\ ディレクトリを、\ ``app``\ ディレクトリの隣にあるdropletのrootに含めます。

==================================================
Buildpack Detection
==================================================
.. _detection:

開発者が、app manifestの中もしくは\ `cf push APP-NAME -b BUILDPACK-NAME`\ cf CLIコマンドの中で、buildpackを指定しなかった場合、Cloud Foundryはbuildpack detectedを実行します。

.. Note::

   Cloud Foundryはappで使用するbuildpackを1つだけ見つけます。

プラットフォームが(buildpackの）探索を実行するときは、以下を実行します:

1. 各buildpackで\ ``/bin/detect``\ を実行します
2. ``/bin/detect``\ スクリプトが終了状態にゼロを返す、最初のbuildpackを選択します
3. もし\ ``/bin/finalize``\ が存在する場合:

  #. ``deps``/``index``\ ディレクトリが存在しない場合は、作成します
  #. ``/bin/supply``\ が存在する場合は、\ ``build``\ 、\ ``cache``\ 、\ ``deps``\ ディレクトリおよびbuildpackの\ ``index``\ と一緒に、\ ``/bin/supply``\ を実行します
  #. ``deps``/``index``\ ディレクトリへの変更を受け入れます
  #. ``build``\ ディレクトリへのあらゆる変更は許可されない場合があります
  #. ``build``\ 、\ ``cache``\ 、\ ``deps``\ ディレクトリへ、\ ``/bin/finalize``\ を実行します
  #. ``build``\ ディレクトリへの変更を受け入れます

4. もし\ ``/bin/finalize``\ が存在しない場合:

  #. ``build``\ および\ ``cache``\ ディレクトリへ\ ``/bin/compile``\ を実行します
  #. ``build``\ ディレクトリへの変更を受け入れます

5. staging informationを決定するために\ ``/bin/release``\ を実行します

このプロセスの最後に、\ ``deps``\ ディレクトリを、\ ``app``\ ディレクトリの隣にあるdropletのrootに含めます。

