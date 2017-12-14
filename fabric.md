# fabric 笔记

1、fab命令引用默认文件名为 fabifile.py ，如果使用非默认文件名称，则需要通过 “-f” 来指定，如：fab -H IP -f filename module

2、fab命令格式为：

    fab [options] <command> [:arg1,arg2=val2,host=foo,hosts='h1;h2',....] ....
    常用参数说明：
    -l：显示定义好的任务函数名；
    -f：指定fab入口文件，默认入口文件名为 fabfile.py
    -g：指定网关（中转）设备，比如堡垒机环境，填写堡垒机IP即可；
    -H：指定目标主机，多台主机用“,”号分隔
    -P：以异步方式运行多主机任务，默认为串行运行
    -R：指定role（角色），以角色名区分不同业务组设备；
    -t：设置设备连接超时时间（秒）
    -T：设置远程主机命令执行超时时间（秒）
    -w：当命令执行失败，发出告警，而非默认中止任务
    例：
    fab -p PASSWORD -H IP,IP,IP -- 'command'
    
3、fabfile的编写：

fabfile的主体由多个自定义的任务函数组成，不同任务函数实现不同的操作逻辑

全局属性设定：

    env.hosts：定义目标主机，可以用IP或主机名表示；env.hosts=['IP','IP'...]
    env.exclude_hosts：排除指定主机；env.exclude_hosts=['IP','IP'...]
    env.user：定义用户；env.user="root"
    env.port：定义目标主机端口；env.port="22"
    env.password：定义密码；env.password='PASSWORD'
    env.passwords：定义不同主机对应不同密码，注意在配置passwords时需配置用户、主机、端口等信息；env.passwords={'USER@IP:PORT':'PASSWORD',.....}
    env.gateway：定义网关（中转，堡垒机）IP；env.gateway='IP'
    env.deploy_release_dir：自定义全局变量，格式： env.+"变量名"；env.deploy_release_dir、env.age、env.sex
    env.roledefs：定义角色分组；env.roledefs={'web':['IP','IP'],'db':['IP','IP']}

引用角色分组时使用Python “修饰符” 的形式进行，角色修饰符下面的任务函数为其作用域。

    @roles('web')
    def webtask()
        run('service nginx start')
    @roles('db')
    def dbtask()
        run('service mysql start')

4、fabric常用API：

    local：执行本地命令；local('uname -s')
    lcd：切换本地目录；lcd('/home')
    cd：切换远程目录；cd('/data/logs')
    run：执行远程命令；run('free -m')
    sudo：sudo方式执行远程命令；sudo('service httpd start')
    put：上传本地文件到远程主机；put('/home/user.info','/data/user.info')
    get：从远程主机下载文件到本地；get('/data/user.info','/home/root.info')
    prompt：获得用户输入信息；prompt('please input user password:')
    confirm：获得提示信息确认；confirm("Tests failed.Continue[Y/N]?")
    reboot：重启远程主机；reboot()
    @task：函数修饰符，标识的函数为fab可调用，非标记对fab不可见，纯业务逻辑；
    @runs_once：函数修饰符，标识的函数只会执行一次，不受多台主机影响

5、示例1：查看本地与远程主机信息

    #!/usr/bin/env python
    from fabric.api import *
    
    env.user = 'root'
    env.hosts = ['192.168.11.136','192.168.11.139']
    env.password = 'fuguang'
    
    # 查看本地系统信息，当有多台主机时只运行一次
    @runs_once
    def local_task():
        local('uname -a')
    
    def remote_task():
        # “with”的作用是让后面的表达式的语句继承当前状态，实现"cd /data/logs && ls -l"的效果
        with cd('/usr/local'):
            run('ls -l')

6、示例2：动态获取远程目录列表

    #!/usr/bin/env python
    from fabric.api import *
    
    env.user = 'root'
    env.hosts = ['192.168.11.136','192.168.11.139']
    env.password = 'fuguang'
    
    # 主机遍历过程中，只有第一台触发此函数
    @runs_once
    def In():
        return prompt("Please input directory name:",default = '/home')
    
    def worktask(dirname):
        run('ls -l ' + dirname)
    
    # 限定只有go函数对fab命令可见，
    @task
    def go():
        getdirname = In()
        worktask(getdirname)

7、示例3：网关模式文件上传与执行

    #!/usr/bin/env python    
    from fabric.api import *
    from fabric.context_managers import *
    from fabric.contrib.console import confirm
    
    env.user = 'root'
    # 定义堡垒机IP，作为文件上传、执行的中转设备
    env.gateway = '192.168.11.139'
    env.hosts = ['192.168.11.136']
    # 如果主机密码不一样，可以通过env.passwords字典变量一一指定
    env.passwords = {
        'root@192.168.11.136:22':'fuguang',
        'root@192.168.11.139:22':'fuguang'
    }
    
    # 定义本地文件路径
    lpackpath = '/home/lnmp.tar.gz'
    # 定义远程路径
    rpackpath = '/tmp/install'
    
    @task
    def put_task():
        run("mkdir -p /tmp/install")
        with settings(warn_only=True):
            # 上传软件包
            result = put(lpackpath,rpackpath)
        if result.failed and not confirm("put file failed, Continue[Y/N]?"):
            abort("Aborting file put task!")
    
    @task
    # 执行远程命令
    def run_task():
        with cd("/tmp/install"):
            run("tar -zxvf lnmp.tar.gz")
            # 使用with继续继承 /tmp/install 目录位置状态
            with cd("lnmp"):
                run("echo 'installing lnmp' > centos.txt")
    
    @task
    def go():
        put_task()
        run_task()
