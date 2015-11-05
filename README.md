# 実行方法
- リソースの用意
```
vagrant up
```

- プロビジョニング
```
ansible-playbook -i test mysql.yml
```

- ログイン
```
#db1
ssh -i ~/.vagrant.d/insecure_private_key vagrant@10.11.22.101
#db2
ssh -i ~/.vagrant.d/insecure_private_key vagrant@10.11.22.102
```

- GROUP REPLICATION 開始
```
mysql -u root -proot

# 最初の1台目のみ
mysql> SET GLOBAL group_replication_bootstrap_group=1;

# 共通
mysql> START GROUP_REPLICATION;

# 最初の1台目のみ
mysql> SET GLOBAL group_replication_bootstrap_group=0;

# 確認
mysql> SHOW VARIABLES LIKE 'group_replication%';
mysql> SELECT * FROM performance_schema.replication_connection_status\G;
mysql> SELECT * FROM performance_schema.replication_group_members;

mysql> exit
```

# 補足
- Vagrant
    - 起動  
      `vagrant up`
    - 停止  
      `vagnrat halt`
    - 削除  
      `vagrant destroy`
    - サンドボックス状態確認 (required: sahara)  
      `vagrant sandbox status`
    - サンドボックス開始 (required: sahara)  
      `vagrant sandbox on`
    - サンドボックスコミット (required: sahara)  
      `vagrant sandbox commit`
    - サンドボックスロールバック (required: sahara)  
      `vagrant sandbox rollback`

- Ansible
    - 実行前の確認  
      `ansible-playbook -i test mysql.yml --list-tasks`
    - 実行  
      `ansible-playbook -i test mysql.yml`

- MySQL
    - Binary Log の見方  
      `mysqlbinlog mysql-bin.xxxxxxxx`

    - 不整合によるエラーのSKIP方法  
      通常のMaster/Slave方式の場合は、`show slave status`で確認して対応することができるのだが、  
      GroupReplicationの起動時にこのエラーが発生する場合は、レコードが存在しないため判断ができない。  
      `show master status`である程度わかるので、それを元にリレーログを参照して判断します。  
      * TroubleShooting参照


# Trouble Shooting
- XComのプロトコルが利用するポートがLISTENしていないため、GROUP REPLICATIONが開始できない。
```
#
# レプリケーションを開始、、、、エラーになります。
#
[vagrant@db2 ~]# mysql -u root -p -e 'START GROUP_REPLICATION;'
ERROR 3096 (HY000) at line 1: The START GROUP_REPLICATION command failed as there was an error when initializing the group communication layer.

#
# ログを確認します。
#
[root@db2 ~]# tailf /var/log/mysqld.log 
（省略）
[XCOM BINDING DEBUG] ::join():: I am NOT the boot node.
[XCOM BINDING DEBUG] ::join():: xcom_client_open_connection to 10.11.22.101:10300
connecting to 10.11.22.101 10300
[XCOM BINDING DEBUG] ::join():: Error on xcom_client_open_connection to 10.11.22.101:10300. Error= -1
connecting to 10.11.22.101 10300
[XCOM BINDING DEBUG] ::join():: Error on xcom_client_open_connection to 10.11.22.101:10300. Error= -1
connecting to 10.11.22.101 10300
（省略）

#
# エラーのキーワードとして、下記のワードがあります。
#   Error on xcom_client_open_connection to 10.11.22.101:10300
#
# どうやら、GROUP REPLICATIONで利用するXComプロトコル用のポートがLISTENしていないみたいです。
# そこで、「10.11.22.101」のLISTENポートを確認してみます。
#
[vagrant@db1 ~]$ sudo netstat -oan |grep LISTEN |grep 10300

#
# やはり、LISTENしていません。
# この場合、GROUP_REPLICATIONの停止/開始を試してみて
# それでも改善しないようなら、再起動をかけます。
#
# 注意点としては、GROUP_REPLICATIONの起動時に必ず
# 「SET GLOBAL group_replication_bootstrap_group=1;」を実行することです。
# これをやらずに「START GROUP_REPLICATION;」を実行するとmysqlクライアントがハングアップします。
# （もしかしたら、retryしているせいなのかもしれませんが、テストをした限りは応答が返ってきませんでした。）
#
[vagrant@db1 ~]# mysql -u root -p -e 'STOP GROUP_REPLICATION;'
[vagrant@db1 ~]# mysql -u root -p -e 'SET GLOBAL group_replication_bootstrap_group=1;'
[vagrant@db1 ~]# mysql -u root -p -e 'START GROUP_REPLICATION;'
[vagrant@db1 ~]# mysql -u root -p -e 'SET GLOBAL group_replication_bootstrap_group=0;'

#
# 問題なく開始できたら、LISTENしているかを確認してみてください。
#
[vagrant@db1 ~]$ netstat -oan |grep 10300
tcp        0      0 0.0.0.0:10300               0.0.0.0:*                   LISTEN      off (0.00/0/0)
tcp        0      0 10.11.22.101:35767          10.11.22.101:10300          ESTABLISHED off (0.00/0/0)
tcp        0      0 10.11.22.101:35776          10.11.22.101:10300          ESTABLISHED off (0.00/0/0)
tcp        0      0 10.11.22.101:10300          10.11.22.101:35769          ESTABLISHED off (0.00/0/0)
tcp        0      0 10.11.22.101:10300          10.11.22.101:35762          ESTABLISHED off (0.00/0/0)
（省略）
```

- 不整合によってリカバリーが失敗し、GROUP REPLICATIONを開始できない
```
#
# レプリケーションを開始、、、、エラーになります。
#
[vagrant@db2 ~]# mysql -u root -p -e 'START GROUP_REPLICATION;'
ERROR 3096 (HY000): The START GROUP_REPLICATION command failed as there was an error when initializing the group communication layer.

#
# ログを確認します。
#
[vagrant@db2 ~]# sudo tailf /var/log/mysqld.log 
（省略）
[XCOM BINDING DEBUG] ::join()
2015-11-05T08:28:18.518445Z 5 [Note] Slave SQL thread for channel 'group_replication_applier' initialized, starting replication in log 'FIRST' at position 0, relay log './db2-relay-bin-group_replication_applier.000005' position: 2136
2015-11-05T08:28:18.518789Z 5 [ERROR] Slave SQL for channel 'group_replication_applier': Error 'Can't drop database 'sample2'; database doesn't exist' on query. Default database: 'sample2'. Query: 'drop database sample2', Error_code: 1008
2015-11-05T08:28:18.518859Z 5 [Warning] Slave: Can't drop database 'sample2'; database doesn't exist Error_code: 1008
2015-11-05T08:28:18.519220Z 5 [ERROR] Error running query, slave SQL thread aborted. Fix the problem, and restart the slave SQL thread with "SLAVE START". We stopped at log 'FIRST' position 0
（省略）

#
# エラーのキーワードとして、下記のワードがあります。
#   ./db2-relay-bin-group_replication_applier.000005
#   drop database sample2
#
# どうやら、GROUP REPLICATIONに参加する前の回復処理で、エラーになっているみたいです。
# ただ、このままだとSKIPしていいGTIDが判断できません。
# 
# そこで、master status情報を確認します。
# とても見にくいですが、3行ある「Executed_Gtid_Set」の列が実行済みのGTIDになります。
#
[vagrant@db2 ~]# mysql -u root -p -e 'SHOW MASTER STATUS'
+-------------------+----------+--------------+------------------+-------------------------------------------------------------------------------------------------------------------------------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                                                                                                                         |
+-------------------+----------+--------------+------------------+-------------------------------------------------------------------------------------------------------------------------------------------+
| master-bin.000003 |      318 |              |                  | 0cc73a76-818d-11e5-bfae-080027079e3d:1-16:18:20-26:28,
d7a30ec1-838d-11e5-b012-080027079e3d:1-2,
d7e15e00-838d-11e5-b17e-080027079e3d:1-2 |
+-------------------+----------+--------------+------------------+-------------------------------------------------------------------------------------------------------------------------------------------+

#
# 怪しいのは、「0cc73a76-818d-11e5-bfae-080027079e3d:1-16:18:20-26:28」
# 1-16と18,20-26を実行しているということは、、、、、、
# SKIPするのは「0cc73a76-818d-11e5-bfae-080027079e3d」の「17」だと予測を立てます。
#
# 念のため、「./db2-relay-bin-group_replication_applier.000005」も確認しておきます。
# lessで見るとメモリが厳しいかもしれないので、別ファイルに出力してからgrepで検索します。
# 一応、前後5行程度も確認しておきます。
#
[vagrant@db2 ~]# mysqlbinlog /var/lib/mysql/db2-relay-bin-group_replication_applier.000005 > /tmp/000005.log
[vagrant@db2 ~]$ grep -e "0cc73a76-818d-11e5-bfae-080027079e3d:17" -5 /tmp/000005.log
/*!*/;
# at 2136
#151105  7:46:23 server id 1  end_log_pos 201 Transaction_context: server_uuid=d7a30ec1-838d-11e5-b012-080027079e3d	thread_id=4
# at 2337
#151105  7:38:21 server id 1  end_log_pos 61 	GTID	last_committed=0	sequence_number=1
SET @@SESSION.GTID_NEXT= '0cc73a76-818d-11e5-bfae-080027079e3d:17'/*!*/;
# at 2398
#151105  7:46:23 server id 1  end_log_pos 87 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1446709583/*!*/;
drop database sample2
/*!*/;

#
# エラーのキーワドと予測を立てたGTIDが一致しました。
#
#     余談ですが、今回はエラーを出すためにわざわざ不正行為を行いました。
#     やり方はGROUP REPLICATIONに参加している他のノードで下記の通りの作業を事前に行っておりました、、、、
#     > STOP GROUP_REPLICATION;
#     > create database sample2;
#     > START GROUP_REPLICATION;
#     > drop database sample2;
#
# 該当のGTIDが判断できたところで、SKIPさせるように指示を出して、再度GROUP REPLICATIONの開始を試してみます。
#
[vagrant@db2 ~]# mysql -u root -p -e "SET GTID_NEXT = '0cc73a76-818d-11e5-bfae-080027079e3d:17'"
[vagrant@db2 ~]# mysql -u root -p -e 'START GROUP_REPLICATION;'
```
