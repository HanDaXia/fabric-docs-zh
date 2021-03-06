
Using CouchDB - 使用 CouchDB
=============

This tutorial will describe the steps required to use the CouchDB as the state
database with Hyperledger Fabric. By now, you should be familiar with Fabric
concepts and have explored some of the samples and tutorials.

本教程将讲述在 Hyperledger Fabric 中使用 CouchDB 作为状态数据库的步骤。现在，
你应该已经熟悉 Fabric 的概念并且已经浏览了一些示例和教程。

The tutorial will take you through the following steps:

本教程将带你按如下步骤与学习：

#. :ref:`cdb-enable-couch`
#. :ref:`cdb-create-index`
#. :ref:`cdb-add-index`
#. :ref:`cdb-install-instantiate`
#. :ref:`cdb-query`
#. :ref:`cdb-pagination`
#. :ref:`cdb-update-index`
#. :ref:`cdb-delete-index`

For a deeper dive into CouchDB refer to :doc:`couchdb_as_state_database`
and for more information on the Fabric ledger refer to the `Ledger <ledger/ledger.html>`_
topic. Follow the tutorial below for details on how to leverage CouchDB in your
blockchain network.

想要更深入的研究 CouchDB 的话，请参阅 :doc:`couchdb_as_state_database` ，关于 Fabric 账
本的跟多信息请参阅 `Ledger <ledger/ledger.html>`_ 主题。下边的教程将详细讲述如何在你的区
块链网络中使用 CouchDB 。

Throughout this tutorial we will use the `Marbles sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__
as our use case to demonstrate how to use CouchDB with Fabric and will deploy
Marbles to the :doc:`build_network` (BYFN) tutorial network. You should have
completed the task :doc:`install`. However, running the BYFN tutorial is not
a prerequisite for this tutorial, instead the necessary commands are provided
throughout this tutorial to use the network.

本教程将使用 `Marbles sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__ 
作为演示在 Fabric 中使用 CouchDB 的用例，并且将会把 Marbles 部署在 :doc:`build_network` （BYFN） 
教程网络上。

Why CouchDB? - 为什么是 CouchDB ？
~~~~~~~~~~~~

Fabric supports two types of peer databases. LevelDB is the default state
database embedded in the peer node and stores chaincode data as simple
key-value pairs and supports key, key range, and composite key queries only.
CouchDB is an optional alternate state database that supports rich
queries when chaincode data values are modeled as JSON. Rich queries are more
flexible and efficient against large indexed data stores, when you want to
query the actual data value content rather than the keys. CouchDB is a JSON
document datastore rather than a pure key-value store therefore enabling
indexing of the contents of the documents in the database.

Fabric 支持两种类型的节点数据库。LevelDB 是默认嵌入在 peer 节点的状态数据库，
用于将链码数据存储为简单的键-值对，仅支持键、键范围和复合键查询。CouchDB 是一
个可选的状态数据库，当链码数据以 JSON 建模时，它支持富查询。当您要查询实际数据
内容而不是键时，富查询对于大型索引数据存储更加灵活和高效。CouchDB 是一个 JSON 
文本数据存储，而不是一个纯键-值存储，并且支持数据库中文本数据的索引。

In order to leverage the benefits of CouchDB, namely content-based JSON
queries,your data must be modeled in JSON format. You must decide whether to use
LevelDB or CouchDB before setting up your network. Switching a peer from using
LevelDB to CouchDB is not supported due to data compatibility issues. All peers
on the network must use the same database type. If you have a mix of JSON and
binary data values, you can still use CouchDB, however the binary values can
only be queried based on key, key range, and composite key queries.

为了发挥 CouchDB 的优势，也就是说基于内容的 JSON 查询，你的数据必须以 JSON 格式
建模。你必须在设置你的网络之前确定使用 LevelDB 还是 CouchDB 。由于数据兼容性的问
题，不支持节点从 LevelDB 切换为 CouchDB 。网络中的所有节点必须使用相同的数据库类
型。如果你想 JSON 和二进制数据混合使用，你同样可以使用 CouchDB ，但是二进制数据只
能根据键、键范围和复合键查询。

.. _cdb-enable-couch:

Enable CouchDB in Hyperledger Fabric - 在 Hyperledger Fabric 中启用 CouchDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CouchDB runs as a separate database process alongside the peer, therefore there
are additional considerations in terms of setup, management, and operations.
A docker image of `CouchDB <https://hub.docker.com/r/hyperledger/fabric-couchdb/>`__
is available and we recommend that it be run on the same server as the
peer. You will need to setup one CouchDB container per peer
and update each peer container by changing the configuration found in
``core.yaml`` to point to the CouchDB container. The ``core.yaml``
file must be located in the directory specified by the environment variable
FABRIC_CFG_PATH:

CouchDB 是独立于节点运行的一个数据库进程，所以在安装、管理和操作的时候有一些额外
的注意事项。有一个可用的 docker 镜像 `CouchDB <https://hub.docker.com/r/hyperledger/fabric-couchdb/>`__ 
并且我们建议它和节点运行在同一个服务器上。我们需要在每一个节点上安装一个 CouchDB 
容器，并且更新每一个节点的配置文件 ``core.yaml`` ，将节点指向 CouchDB 容器。 
``core.yaml`` 文件的路径必须在环境变量 FABRIC_CFG_PATH 中指定：


* For docker deployments, ``core.yaml`` is pre-configured and located in the peer
  container ``FABRIC_CFG_PATH`` folder. However when using docker environments,
  you typically pass environment variables by editing the
  ``docker-compose-couch.yaml``  to override the core.yaml

* 对于 docker 的部署，在节点容器中 ``FABRIC_CFG_PATH`` 指定的文件夹中的 ``core.yaml`` 
  是预先配置好的。如果你要使用 docker 环境，你可以通过重写 ``docker-compose-couch.yaml`` 
  中的环境变量来覆盖 core.yaml 

* For native binary deployments, ``core.yaml`` is included with the release artifact
  distribution.

* 对于原生的二进制部署， ``core.yaml`` 包含在发布的构件中。

Edit the ``stateDatabase`` section of ``core.yaml``. Specify ``CouchDB`` as the
``stateDatabase`` and fill in the associated ``couchDBConfig`` properties. For
more details on configuring CouchDB to work with fabric, refer `here <http://hyperledger-fabric.readthedocs.io/en/master/couchdb_as_state_database.html#couchdb-configuration>`__.
To view an example of a core.yaml file configured for CouchDB, examine the
BYFN ``docker-compose-couch.yaml`` in the ``HyperLedger/fabric-samples/first-network``
directory.

编辑 ``core.yaml`` 中的 ``stateDatabase`` 部分。将 ``stateDatabase`` 指定为 ``CouchDB`` 
并且填写 ``couchDBConfig`` 相关的配置。在 Fabric 中配置 CouchDB 的更多细节，请参阅 
`这里 <http://hyperledger-fabric.readthedocs.io/en/master/couchdb_as_state_database.html#couchdb-configuration>`__ 。
配置 CouchDB 的示例 core.yaml 文件，请查看 ``HyperLedger/fabric-samples/first-network`` 
文件夹下 BYFN 的 ``docker-compose-couch.yaml`` 。

.. _cdb-create-index:

Create an index - 创建一个索引
~~~~~~~~~~~~~~~

Why are indexes important?

为什么索引很重要？

Indexes allow a database to be queried without having to examine every row
with every query, making them run faster and more efficiently. Normally,
indexes are built for frequently occurring query criteria allowing the data to
be queried more efficiently. To leverage the major benefit of CouchDB -- the
ability to perform rich queries against JSON data -- indexes are not required,
but they are strongly recommended for performance. Also, if sorting is required
in a query, CouchDB requires an index of the sorted fields.

索引可以让数据库不用在每次查询的时候都检查每一行，可以让数据库运行的更快和更高效。
一般来说，对频繁查询的数据进行索引可以使数据查询更高效。为了充分发挥 CouchDB 的优
势 -- 对 JSON 数据进行富查询的能力 -- 并不需要索引，但是为了性能考虑强烈建议建立
索引。另外，如果在一个查询中需要排序，CouchDB 需要在排序的字段有一个索引。

.. note::

   Rich queries that do not have an index will work but may throw a warning
   in the CouchDB log that the index was not found. However, if a rich query
   includes a sort specification, then an index on that field is required;
   otherwise, the query will fail and an error will be thrown.

.. note::

   没有索引的情况下富查询也是可以使用的，但是会在 CouchDB 的日志中抛出一个没
   有找到索引的警告。如果一个富查询中包含了一个排序的说明，需要排序的那个字段
   就必须有索引；否则，查询将会失败并抛出错误。

To demonstrate building an index, we will use the data from the `Marbles
sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__.
In this example, the Marbles data structure is defined as:

为了演示创建一个索引，我们将使用 `Marbles sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__ 
中的数据。在本例中，Marbles 的数据结构定义为：

.. code:: javascript

  type marble struct {
	   ObjectType string `json:"docType"` //docType is used to distinguish the various types of objects in state database
	   Name       string `json:"name"`    //the field tags are needed to keep case from bouncing around
	   Color      string `json:"color"`
           Size       int    `json:"size"`
           Owner      string `json:"owner"`
  }

In this structure, the attributes (``docType``, ``name``, ``color``, ``size``,
``owner``) define the ledger data associated with the asset. The attribute
``docType`` is a pattern used in the chaincode to differentiate different data
types that may need to be queried separately. When using CouchDB, it
recommended to include this ``docType`` attribute to distinguish each type of
document in the chaincode namespace. (Each chaincode is represented as its own
CouchDB database, that is, each chaincode has its own namespace for keys.)

在这个结构体中，（ ``docType``, ``name``, ``color``, ``size``, ``owner`` ）属性
定义了和资产相关的账本数据。 ``docType`` 属性用来在链码中区分可能需要单独查询的
不同数据类型的模式。当时使用 CouchDB 的时候，建议包含 ``docType`` 属性来区分在链
码命名空间中的每一个文档。（每一个链码都需要有他们自己的 CouchDB 数据库，也就是
说，每一个链码都有它自己的键的命名空间。）

With respect to the Marbles data structure, ``docType`` is used to identify
that this document/asset is a marble asset. Potentially there could be other
documents/assets in the chaincode database. The documents in the database are
searchable against all of these attribute values.

在 Marbles 数据结构的定义中， ``docType`` 用来识别这个文档或者资产是一个弹珠资产。
同时在链码数据库中也可能存在其他文档或者资产。数据库中的文档对于这些属性值来说都是
可查询的。

When defining an index for use in chaincode queries, each one must be defined
in its own text file with the extension `*.json` and the index definition must
be formatted in the CouchDB index JSON format.

当为链码查询定义一个索引的时候，每一个索引都必须定义在一个扩展名为 ``*.json`` 的
文本文件中，并且索引定义的格式必须为 CouchDB 索引的 JSON 格式。

To define an index, three pieces of information are required:

需要以下三条信息来定义一个索引：

  * `fields`: these are the frequently queried fields
  * `fields`: 这些是常用的查询字段
  * `name`: name of the index
  * `name`: 索引名
  * `type`: always json in this context
  * `type`: 它的内容一般是 json

For example, a simple index named ``foo-index`` for a field named ``foo``.

例如，这是一个对字段 ``foo`` 的一个名为 ``foo-index`` 的简单索引。

.. code:: json

    {
        "index": {
            "fields": ["foo"]
        },
        "name" : "foo-index",
        "type" : "json"
    }

Optionally the design document  attribute ``ddoc`` can be specified on the index
definition. A `design document <http://guide.couchdb.org/draft/design.html>`__ is
CouchDB construct designed to contain indexes. Indexes can be grouped into
design documents for efficiency but CouchDB recommends one index per design
document.

可选地，设计文档（ design document ）属性 ``ddoc`` 可以写在索引的定义中。 
`design document <http://guide.couchdb.org/draft/design.html>`__ 是 CouchDB 结构,
用于包含索引。索引可以以组的形式定义在设计文档中以提升效率，但是 CouchDB 建议每一
个设计文档包含一个索引。

.. tip:: When defining an index it is a good practice to include the ``ddoc``
         attribute and value along with the index name. It is important to
         include this attribute to ensure that you can update the index later
         if needed. Also it gives you the ability to explicitly specify which
         index to use on a query.

.. tip:: 当定义一个索引的时候，最好将 ``ddoc`` 属性和值包含在索引内。包含这个
         属性以确保在你需要的时候升级索引，这是很重要的。它还使你能够明确指定
         要在查询上使用的索引。

Here is another example of an index definition from the Marbles sample with
the index name ``indexOwner`` using multiple fields ``docType`` and ``owner``
and includes the ``ddoc`` attribute:

这里有另外一个使用 Marbles 示例定义索引的例子，在索引 ``indexOwner`` 使用了多个
字段 ``docType`` 和 ``owner`` 并且包含了 ``ddoc`` 属性：

.. _indexExample:

.. code:: json

  {
    "index":{
        "fields":["docType","owner"] // Names of the fields to be queried
    },
    "ddoc":"indexOwnerDoc", // (optional) Name of the design document in which the index will be created.
    "name":"indexOwner",
    "type":"json"
  }

In the example above, if the design document ``indexOwnerDoc`` does not already
exist, it is automatically created when the index is deployed. An index can be
constructed with one or more attributes specified in the list of fields and
any combination of attributes can be specified. An attribute can exist in
multiple indexes for the same docType. In the following example, ``index1``
only includes the attribute ``owner``, ``index2`` includes the attributes
``owner and color`` and ``index3`` includes the attributes ``owner, color and
size``. Also, notice each index definition has its own ``ddoc`` value, following
the CouchDB recommended practice.

在上边的例子中，如果设计文档 ``indexOwnerDoc`` 不存在，当索引部署的时候会自动创建
一个。一个索引可以根据字段列表中指定的一个或者多个属性构建，而且可以定义任何属性的
组合。一个属性可以存在于同一个 docType 的多个索引中。在下边的例子中， ``index1`` 
只包含 ``owner`` 属性， ``index2`` 包含 ``owner 和 color`` 属性， ``index3`` 包含 
``owner、 color 和 size`` 属性。另外，注意，根据 CouchDB 的建议，每一个索引的定义
都包含一个它们自己的 ``ddoc`` 值。

.. code:: json

  {
    "index":{
        "fields":["owner"] // Names of the fields to be queried
    },
    "ddoc":"index1Doc", // (optional) Name of the design document in which the index will be created.
    "name":"index1",
    "type":"json"
  }

  {
    "index":{
        "fields":["owner", "color"] // Names of the fields to be queried
    },
    "ddoc":"index2Doc", // (optional) Name of the design document in which the index will be created.
    "name":"index2",
    "type":"json"
  }

  {
    "index":{
        "fields":["owner", "color", "size"] // Names of the fields to be queried
    },
    "ddoc":"index3Doc", // (optional) Name of the design document in which the index will be created.
    "name":"index3",
    "type":"json"
  }


In general, you should model index fields to match the fields that will be used
in query filters and sorts. For more details on building an index in JSON
format refer to the `CouchDB documentation <http://docs.couchdb.org/en/latest/api/database/find.html#db-index>`__.

一般来说，你为索引字段建模应该匹配将用于查询过滤和排序的字段。对于以 JSON 格式
构建索引的更多信息请参阅 `CouchDB documentation <http://docs.couchdb.org/en/latest/api/database/find.html#db-index>`__ 。

A final word on indexing, Fabric takes care of indexing the documents in the
database using a pattern called ``index warming``. CouchDB does not typically
index new or updated documents until the next query. Fabric ensures that
indexes stay 'warm' by requesting an index update after every block of data is
committed.  This ensures queries are fast because they do not have to index
documents before running the query. This process keeps the index current
and refreshed every time new records are added to the state database.

关于索引最后要说的是，Fabric 在数据库中为文档建立索引的时候使用一种成为 ``索引升温
（index warming）`` 的模式。 CouchDB 直到下一次查询的时候才会索引新的或者更新的
文档。Fabric 通过在每一个数据区块提交完之后请求索引更新的方式，来确保索引处于 ‘热
（warm）’ 状态。这就确保了查询速度快，因为在运行查询之前不用索引文档。这个过程保
持了索引的现状，并在每次新数据添加到状态数据的时候刷新。

.. _cdb-add-index:


Add the index to your chaincode folder - 将索引添加到你的链码文件夹
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once you finalize an index, it is ready to be packaged with your chaincode for
deployment by being placed alongside it in the appropriate metadata folder.

当你完成索引之后，你需要把它打包到你的链码中，以便于将它部署到合适的元数据文件夹。

If your chaincode installation and instantiation uses the Hyperledger
Fabric Node SDK, the JSON index files can be located in any folder as long
as it conforms to this `directory structure <https://fabric-sdk-node.github.io/tutorial-metadata-chaincode.html>`__.
During the chaincode installation using the client.installChaincode() API,
include the attribute (``metadataPath``) in the `installation request <https://fabric-sdk-node.github.io/global.html#ChaincodeInstallRequest>`__.
The value of the metadataPath is a string representing the absolute path to the
directory structure containing the JSON index file(s).

如果你使用 Hyperledger Fabric Node SDK 安装和初始化链码，JSON 索引文件可以放在符
合这个 `目录结构<https://fabric-sdk-node.github.io/tutorial-metadata-chaincode.html>`__ 
的任意位置。在使用 client.installChaincode() API 安装链码的时候，需要包含在 
`安装请求 <https://fabric-sdk-node.github.io/global.html#ChaincodeInstallRequest>`__ 
中的属性 （ ``metadataPath`` ）。

Alternatively, if you are using the
:doc:`peer-commands` to install and instantiate the chaincode, then the JSON
index files must be located under the path ``META-INF/statedb/couchdb/indexes``
which is located inside the directory where the chaincode resides.

或者，如果你使用 :doc:`peer-commands` 安装和初始化链码， JSON 索引文件必须放在
链码所在目录的 ``META-INF/statedb/couchdb/indexes`` 路径下。

The `Marbles sample <https://github.com/hyperledger/fabric-samples/tree/master/chaincode/marbles02/go>`__  below illustrates how the index
is packaged with the chaincode which will be installed using the peer commands.

下边的 `Marbles 示例 <https://github.com/hyperledger/fabric-samples/tree/master/chaincode/marbles02/go>`__ 
说明了如何使用 peer 命令将索引打包进将要安装的链码中。

.. image:: images/couchdb_tutorial_pkg_example.png
  :scale: 100%
  :align: center
  :alt: Marbles Chaincode Index Package


Start the network - 启动网络
-----------------

 :guilabel:`Try it yourself`

 Before installing and instantiating the marbles chaincode, we need to start
 up the BYFN network. For the sake of this tutorial, we want to operate
 from a known initial state. The following command will kill any active
 or stale docker containers and remove previously generated artifacts.
 Therefore let's run the following command to clean up any
 previous environments:

 在安装和初始化弹珠链码之前，我们需要启动 BYFN 网络。考虑到本教程的目的，
 我们需要在一个已知的初始状态操作。下边的命令将关闭所有激活状态或者存在
 的 docker 容器，并删除之前生成的构建。然后，我们运行下边的命令来清除所
 有之前的环境：

 .. code:: bash

  cd fabric-samples/first-network
  ./byfn.sh down


 Now start up the BYFN network with CouchDB by running the following command:

 现在使用下边的命令启动启用了 CouchDB 的 BYFN 网络：
 
 .. code:: bash

   ./byfn.sh up -c mychannel -s couchdb

 This will create a simple Fabric network consisting of a single channel named
 `mychannel` with two organizations (each maintaining two peer nodes) and an
 ordering service while using CouchDB as the state database.

 这将创建一个简单的 Fabric 网络，其中包含一个叫 `mychannel` 的通道，通道中
 有两个组织（每个组织两个 peer 节点）和一个排序服务，同时使用 CouchDB 作为
 状态数据库。

.. _cdb-install-instantiate:

Install and instantiate the Chaincode - 安装和初始化链码
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Client applications interact with the blockchain ledger through chaincode. As
such we need to install the chaincode on every peer that will
execute and endorse our transactions and instantiate the chaincode on the
channel. In the previous section, we demonstrated how to package the chaincode
so they should be ready for deployment.

客户端应用通过链码和区块链账本交互。所以我们需要在每一个执行和背书交易的节点
上安装链码，并且在通道上初始化链码。在之前的章节中，我们演示了如何打包链码，
所以他们应该已经准备好部署了。

Chaincode is installed onto a peer and then instantiated onto the channel using
:doc:`peer-commands`.

我们将使用 :doc:`peer-commands` 将链码安装到节点，然后在通道上初始化。

1. Use the `peer chaincode install <http://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20instantiate#peer-chaincode-install>`__ command to install the Marbles chaincode on a peer.

1. 使用`peer chaincode install <http://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20instantiate#peer-chaincode-install>`__ 
  命令将链码安装到节点上。

 :guilabel:`Try it yourself`

 Assuming you have started the BYFN network, navigate into the CLI
 container using the command:

 假设你已经启动了 BYFN 网路，使用下边的命令进入到 CLI 容器：

 .. code:: bash

      docker exec -it cli bash

 Use the following command to install the Marbles chaincode from the git
 repository onto a peer in your BYFN network. The CLI container defaults
 to using peer0 of org1:

 使用下边命令从 github 仓库将 Marbles 链码安装到你的 BYFN 网络。CLI 容器
 默认使用 org1 的 peer0 节点：

 .. code:: bash

      peer chaincode install -n marbles -v 1.0 -p github.com/chaincode/marbles02/go

2. Issue the `peer chaincode instantiate <http://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20instantiate#peer-chaincode-instantiate>`__ command to instantiate the
chaincode on a channel.

2. 执行 `peer chaincode instantiate <http://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20instantiate#peer-chaincode-instantiate>`__  
   命令在通道上初始化链码。

 :guilabel:`Try it yourself`

 To instantiate the Marbles sample on the BYFN channel ``mychannel``
 run the following command:

 使用下边的命令在 BYFN 通道 ``mychannel`` 上初始化 Marbles 示例：

 .. code:: bash

    export CHANNEL_NAME=mychannel
    peer chaincode instantiate -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -v 1.0 -c '{"Args":["init"]}' -P "OR ('Org0MSP.peer','Org1MSP.peer')"

Verify index was deployed - 验证部署的索引
-------------------------

Indexes will be deployed to each peer's CouchDB state database once the
chaincode is both installed on the peer and instantiated on the channel. You
can verify that the CouchDB index was created successfully by examining the
peer log in the Docker container.

当链码在节点上安装并且在通道上实例化完成之后，索引会被部署到每一个节点的 CouchDB 
状态数据库上。你可以通过检查 Docker 容器中的节点日志来确认 CouchDB 是否被创建成功。

 :guilabel:`Try it yourself`

 To view the logs in the peer docker container,
 open a new Terminal window and run the following command to grep for message
 confirmation that the index was created.

 为了查看节点 docker 容器的日志，打开一个新的终端窗口，然后运行下边的命令来匹配索
 引被创建的确认信息。

 ::

   docker logs peer0.org1.example.com  2>&1 | grep "CouchDB index"


 You should see a result that looks like the following:

 你将会看到类似下边的结果：

 ::

    [couchdb] CreateIndex -> INFO 0be Created CouchDB index [indexOwner] in state database [mychannel_marbles] using design document [_design/indexOwnerDoc]

 .. note:: If Marbles was not installed on the BYFN peer ``peer0.org1.example.com``,
          you may need to replace it with the name of a different peer where
          Marbles was installed.

 .. note:: 如果 Marbles 没有安装在 BYFN 的节点 ``peer0.org1.example.com`` 上，你可
           能需要切换到其他的安装了 Marbles 的节点。

.. _cdb-query:

Query the CouchDB State Database - 查询 CouchDB 状态数据库
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that the index has been defined in the JSON file and deployed alongside
the chaincode, chaincode functions can execute JSON queries against the CouchDB
state database, and thereby peer commands can invoke the chaincode functions.

Specifying an index name on a query is optional. If not specified, and an index
already exists for the fields being queried, the existing index will be
automatically used.

.. tip:: It is a good practice to explicitly include an index name on a
         query using the ``use_index`` keyword. Without it, CouchDB may pick a
         less optimal index. Also CouchDB may not use an index at all and you
         may not realize it, at the low volumes during testing. Only upon
         higher volumes you may realize slow performance because CouchDB is not
         using an index and you assumed it was.


Build the query in chaincode
----------------------------

You can perform complex rich queries against the chaincode data values using
the CouchDB JSON query language within chaincode. As we explored above, the
`marbles02 sample chaincode <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__
includes an index and rich queries are defined in the functions - ``queryMarbles``
and ``queryMarblesByOwner``:

  * **queryMarbles** --

      Example of an **ad hoc rich query**. This is a query
      where a (selector) string can be passed into the function. This query
      would be useful to client applications that need to dynamically build
      their own selectors at runtime. For more information on selectors refer
      to `CouchDB selector syntax <http://docs.couchdb.org/en/latest/api/database/find.html#find-selectors>`__.



  * **queryMarblesByOwner** --

      Example of a parameterized query where the
      query logic is baked into the chaincode. In this case the function accepts
      a single argument, the marble owner. It then queries the state database for
      JSON documents matching the docType of “marble” and the owner id using the
      JSON query syntax.


Run the query using the peer command
------------------------------------

In absence of a client application to test rich queries defined in chaincode,
peer commands can be used. Peer commands run from the command line inside the
docker container. We will customize the `peer chaincode query <http://hyperledger-fabric.readthedocs.io/en/master/commands/peerchaincode.html?%20chaincode%20query#peer-chaincode-query>`__
command to use the Marbles index ``indexOwner`` and query for all marbles owned
by "tom" using the ``queryMarbles`` function.

 :guilabel:`Try it yourself`

 Before querying the database, we should add some data. Run the following
 command in the peer container to create a marble owned by "tom":

 .. code:: bash

   peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble1","blue","35","tom"]}'

 After an index has been deployed during chaincode instantiation, it will
 automatically be utilized by chaincode queries. CouchDB can determine which
 index to use based on the fields being queried. If an index exists for the
 query criteria it will be used. However the recommended approach is to
 specify the ``use_index`` keyword on the query. The peer command below is an
 example of how to specify the index explicitly in the selector syntax by
 including the ``use_index`` keyword:

 .. code:: bash

   // Rich Query with index name explicitly specified:
   peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["queryMarbles", "{\"selector\":{\"docType\":\"marble\",\"owner\":\"tom\"}, \"use_index\":[\"_design/indexOwnerDoc\", \"indexOwner\"]}"]}'

Delving into the query command above, there are three arguments of interest:

*  ``queryMarbles``
  Name of the function in the Marbles chaincode. Notice a `shim <https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim>`__
  ``shim.ChaincodeStubInterface`` is used to access and modify the ledger. The
  ``getQueryResultForQueryString()`` passes the queryString to the shim API ``getQueryResult()``.

.. code:: bash

  func (t *SimpleChaincode) queryMarbles(stub shim.ChaincodeStubInterface, args []string) pb.Response {

	  //   0
	  // "queryString"
	   if len(args) < 1 {
		   return shim.Error("Incorrect number of arguments. Expecting 1")
	   }

	   queryString := args[0]

	   queryResults, err := getQueryResultForQueryString(stub, queryString)
	   if err != nil {
		 return shim.Error(err.Error())
	   }
	   return shim.Success(queryResults)
  }

*  ``{"selector":{"docType":"marble","owner":"tom"}``
  This is an example of an **ad hoc selector** string which finds all documents
  of type ``marble`` where the ``owner`` attribute has a value of ``tom``.


*  ``"use_index":["_design/indexOwnerDoc", "indexOwner"]``
  Specifies both the design doc name  ``indexOwnerDoc`` and index name
  ``indexOwner``. In this example the selector query explicitly includes the
  index name, specified by using the ``use_index`` keyword. Recalling the
  index definition above :ref:`cdb-create-index`, it contains a design doc,
  ``"ddoc":"indexOwnerDoc"``. With CouchDB, if you plan to explicitly include
  the index name on the query, then the index definition must include the
  ``ddoc`` value, so it can be referenced with the ``use_index`` keyword.


The query runs successfully and the index is leveraged with the following results:

.. code:: json

  Query Result: [{"Key":"marble1", "Record":{"color":"blue","docType":"marble","name":"marble1","owner":"tom","size":35}}]

.. _cdb-pagination:


Query the CouchDB State Database With Pagination
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When large result sets are returned by CouchDB queries, a set of APIs is
available which can be called by chaincode to paginate the list of results.
Pagination provides a mechanism to partition the result set by
specifying a ``pagesize`` and a start point -- a ``bookmark`` which indicates
where to begin the result set. The client application iteratively invokes the
chaincode that executes the query until no more results are returned. For more information refer to
this `topic on pagination with CouchDB <http://hyperledger-fabric.readthedocs.io/en/master/couchdb_as_state_database.html#couchdb-pagination>`__.


We will use the  `Marbles sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__
function ``queryMarblesWithPagination`` to  demonstrate how
pagination can be implemented in chaincode and the client application.

* **queryMarblesWithPagination** --

    Example of an **ad hoc rich query with pagination**. This is a query
    where a (selector) string can be passed into the function similar to the
    above example.  In this case, a ``pageSize`` is also included with the query as
    well as a ``bookmark``.

In order to demonstrate pagination, more data is required. This example assumes
that you have already added marble1 from above. Run the following commands in
the peer container to create four more marbles owned by "tom", to create a
total of five marbles owned by "tom":

:guilabel:`Try it yourself`

.. code:: bash

  peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble2","yellow","35","tom"]}'
  peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble3","green","20","tom"]}'
  peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble4","purple","20","tom"]}'
  peer chaincode invoke -o orderer.example.com:7050 --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C $CHANNEL_NAME -n marbles -c '{"Args":["initMarble","marble5","blue","40","tom"]}'

In addition to the arguments for the query in the previous example,
queryMarblesWithPagination adds ``pagesize`` and ``bookmark``. ``PageSize``
specifies the number of records to return per query.  The ``bookmark`` is an
"anchor" telling couchDB where to begin the page. (Each page of results returns
a unique bookmark.)

*  ``queryMarblesWithPagination``
  Name of the function in the Marbles chaincode. Notice a `shim <https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim>`__
  ``shim.ChaincodeStubInterface`` is used to access and modify the ledger. The
  ``getQueryResultForQueryStringWithPagination()`` passes the queryString along
    with the pagesize and bookmark to the shim API ``GetQueryResultWithPagination()``.

.. code:: bash

  func (t *SimpleChaincode) queryMarblesWithPagination(stub shim.ChaincodeStubInterface, args []string) pb.Response {

  	//   0
  	// "queryString"
  	if len(args) < 3 {
  		return shim.Error("Incorrect number of arguments. Expecting 3")
  	}

  	queryString := args[0]
  	//return type of ParseInt is int64
  	pageSize, err := strconv.ParseInt(args[1], 10, 32)
  	if err != nil {
  		return shim.Error(err.Error())
  	}
  	bookmark := args[2]

  	queryResults, err := getQueryResultForQueryStringWithPagination(stub, queryString, int32(pageSize), bookmark)
  	if err != nil {
  		return shim.Error(err.Error())
  	}
  	return shim.Success(queryResults)
  }


The following example is a peer command which calls queryMarblesWithPagination
with a pageSize of ``3`` and no bookmark specified.

.. tip:: When no bookmark is specified, the query starts with the "first"
         page of records.

:guilabel:`Try it yourself`

.. code:: bash

  // Rich Query with index name explicitly specified and a page size of 3:
  peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["queryMarblesWithPagination", "{\"selector\":{\"docType\":\"marble\",\"owner\":\"tom\"}, \"use_index\":[\"_design/indexOwnerDoc\", \"indexOwner\"]}","3",""]}'

The following response is received (carriage returns added for clarity), three
of the five marbles are returned because the ``pagsize`` was set to ``3``:

.. code:: bash

  [{"Key":"marble1", "Record":{"color":"blue","docType":"marble","name":"marble1","owner":"tom","size":35}},
   {"Key":"marble2", "Record":{"color":"yellow","docType":"marble","name":"marble2","owner":"tom","size":35}},
   {"Key":"marble3", "Record":{"color":"green","docType":"marble","name":"marble3","owner":"tom","size":20}}]
  [{"ResponseMetadata":{"RecordsCount":"3",
  "Bookmark":"g1AAAABLeJzLYWBgYMpgSmHgKy5JLCrJTq2MT8lPzkzJBYqz5yYWJeWkGoOkOWDSOSANIFk2iCyIyVySn5uVBQAGEhRz"}}]

.. note::  Bookmarks are uniquely generated by CouchDB for each query and
           represent a placeholder in the result set. Pass the
           returned bookmark on the subsequent iteration of the query to
           retrieve the next set of results.

The following is a peer command to call queryMarblesWithPagination with a
pageSize of ``3``. Notice this time, the query includes the bookmark returned
from the previous query.

:guilabel:`Try it yourself`

.. code:: bash

  peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["queryMarblesWithPagination", "{\"selector\":{\"docType\":\"marble\",\"owner\":\"tom\"}, \"use_index\":[\"_design/indexOwnerDoc\", \"indexOwner\"]}","3","g1AAAABLeJzLYWBgYMpgSmHgKy5JLCrJTq2MT8lPzkzJBYqz5yYWJeWkGoOkOWDSOSANIFk2iCyIyVySn5uVBQAGEhRz"]}'

The following response is received (carriage returns added for clarity).  The
last two records are retrieved:

.. code:: bash

  [{"Key":"marble4", "Record":{"color":"purple","docType":"marble","name":"marble4","owner":"tom","size":20}},
   {"Key":"marble5", "Record":{"color":"blue","docType":"marble","name":"marble5","owner":"tom","size":40}}]
  [{"ResponseMetadata":{"RecordsCount":"2",
  "Bookmark":"g1AAAABLeJzLYWBgYMpgSmHgKy5JLCrJTq2MT8lPzkzJBYqz5yYWJeWkmoKkOWDSOSANIFk2iCyIyVySn5uVBQAGYhR1"}}]

The final command is a peer command to call queryMarblesWithPagination with
a pageSize of ``3`` and with the bookmark from the previous query.

:guilabel:`Try it yourself`

.. code:: bash

    peer chaincode query -C $CHANNEL_NAME -n marbles -c '{"Args":["queryMarblesWithPagination", "{\"selector\":{\"docType\":\"marble\",\"owner\":\"tom\"}, \"use_index\":[\"_design/indexOwnerDoc\", \"indexOwner\"]}","3","g1AAAABLeJzLYWBgYMpgSmHgKy5JLCrJTq2MT8lPzkzJBYqz5yYWJeWkmoKkOWDSOSANIFk2iCyIyVySn5uVBQAGYhR1"]}'

The following response is received (carriage returns added for clarity).
No records are returned, indicating that all pages have been retrieved:

.. code:: bash

    []
    [{"ResponseMetadata":{"RecordsCount":"0",
    "Bookmark":"g1AAAABLeJzLYWBgYMpgSmHgKy5JLCrJTq2MT8lPzkzJBYqz5yYWJeWkmoKkOWDSOSANIFk2iCyIyVySn5uVBQAGYhR1"}}]

For an example of how a client application can iterate over
the result sets using pagination, search for the ``getQueryResultForQueryStringWithPagination``
function in the `Marbles sample <https://github.com/hyperledger/fabric-samples/blob/master/chaincode/marbles02/go/marbles_chaincode.go>`__.

.. _cdb-update-index:

Update an Index
~~~~~~~~~~~~~~~

It may be necessary to update an index over time. The same index may exist in
subsequent versions of the chaincode that gets installed. In order for an index
to be updated, the original index definition must have included the design
document ``ddoc`` attribute and an index name. To update an index definition,
use the same index name but alter the index definition. Simply edit the index
JSON file and add or remove fields from the index. Fabric only supports the
index type JSON, changing the index type is not supported. The updated
index definition gets redeployed to the peer’s state database when the chaincode
is installed and instantiated. Changes to the index name or ``ddoc`` attributes
will result in a new index being created and the original index remains
unchanged in CouchDB until it is removed.

.. note:: If the state database has a significant volume of data, it will take
          some time for the index to be re-built, during which time chaincode
          invokes that issue queries may fail or timeout.

Iterating on your index definition
----------------------------------

If you have access to your peer's CouchDB state database in a development
environment, you can iteratively test various indexes in support of
your chaincode queries. Any changes to chaincode though would require
redeployment. Use the `CouchDB Fauxton interface <http://docs.couchdb.org/en/latest/fauxton/index.html>`__ or a command
line curl utility to create and update indexes.

.. note:: The Fauxton interface is a web UI for the creation, update, and
          deployment of indexes to CouchDB. If you want to try out this
          interface, there is an example of the format of the Fauxton version
          of the index in Marbles sample. If you have deployed the BYFN network
          with CouchDB, the Fauxton interface can be loaded by opening a browser
          and navigating to ``http://localhost:5984/_utils``.

Alternatively, if you prefer not use the Fauxton UI, the following is an example
of a curl command which can be used to create the index on the database
``mychannel_marbles``:

     // Index for docType, owner.
     // Example curl command line to define index in the CouchDB channel_chaincode database

.. code:: bash

   curl -i -X POST -H "Content-Type: application/json" -d
          "{\"index\":{\"fields\":[\"docType\",\"owner\"]},
            \"name\":\"indexOwner\",
            \"ddoc\":\"indexOwnerDoc\",
            \"type\":\"json\"}" http://hostname:port/mychannel_marbles/_index

.. note:: If you are using BYFN configured with CouchDB, replace hostname:port
	  with ``localhost:5984``.

.. _cdb-delete-index:

Delete an Index
~~~~~~~~~~~~~~~

Index deletion is not managed by Fabric tooling. If you need to delete an index,
manually issue a curl command against the database or delete it using the
Fauxton interface.

The format of the curl command to delete an index would be:

.. code:: bash

   curl -X DELETE http://localhost:5984/{database_name}/_index/{design_doc}/json/{index_name} -H  "accept: */*" -H  "Host: localhost:5984"


To delete the index used in this tutorial, the curl command would be:

.. code:: bash

   curl -X DELETE http://localhost:5984/mychannel_marbles/_index/indexOwnerDoc/json/indexOwner -H  "accept: */*" -H  "Host: localhost:5984"



.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
