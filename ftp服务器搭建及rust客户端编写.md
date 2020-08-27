#### 搭建ftp服务器
本篇采用 vsftpd来搭建，vsftpd在许多的发行版当中成为了默认的配置,系统为 ubuntu

安装

    sudo apt install vsftpd

    // 查看安装是否成功
    vsftpd -v


检查防火墙
> ftp一般需要20,21端口开放 ;
> 安全传输需要 TLS 需要端口，暂定 990 ;
> ftp被动模式需要开发的端口，暂定 40000-50000 ;

    // 检查防火墙状态
    sudo ufw status

    // 没有开放，开放以上配置端口
    sudo ufw allow 20/tcp
    sudo ufw allow 21/tcp
    sudo ufw allow 990/tcp
    sudo ufw allow 40000:50000/tcp


准备用户目录(ftp的根目录)

    // 添加用户,按照提示设置密码等操作
    sudo adduser ftpuser01

    // 创建在该用户下创建一个ftp目录，做为根目录，在后面ftp限制只能访问该目录
    sudo mkdir /home/ftpuser01/ftp

    // 设置所有权
    sudo chown nobody:nogroup /home/ftpuser01/ftp

    // 可以删除某一权限，如写权限
    sudo chmod a-w /home/ftpuser01/ftp

    //查看权限
    sudo ls -la /home/ftpuser01/ftp

    // 创建一个文件上传的目录，并分配权限
    sudo mkdir /home/ftpuser01/ftp/files

    // 赋予权限 ftpuser01
    sudo chown ftpuser01:ftpuser01 /home/ftpuser01/ftp/files

    // 添加一个文件测试用
    echo "vsftpd test file" | sudo tee /home/ftpuser01/ftp/files/test.txt

 配置ftp

    // 备份原配置文件，这个东西最好先操作
    sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak

    // vsftpd.conf 配置示例

    # Example config file /etc/vsftpd.conf
    #
    # The default compiled in settings are fairly paranoid. This sample file
    # loosens things up a bit, to make the ftp daemon more usable.
    # Please see vsftpd.conf.5 for all compiled in defaults.
    #
    # READ THIS: This example file is NOT an exhaustive list of vsftpd options.
    # Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
    # capabilities.
    #
    #
    # Run standalone?  vsftpd can run either from an inetd or as a standalone
    # daemon started from an initscript.
    listen=NO
    #
    # This directive enables listening on IPv6 sockets. By default, listening
    # on the IPv6 "any" address (::) will accept connections from both IPv6
    # and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
    # sockets. If you want that (perhaps because you want to listen on specific
    # addresses) then you must run two copies of vsftpd with two configuration
    # files.
    listen_ipv6=YES
    #
    # Allow anonymous FTP? (Disabled by default).
    
    # 禁止匿名登录
    anonymous_enable=NO
    #
    # Uncomment this to allow local users to log in.
    
    # 允许本地用户登录
    local_enable=YES
    #
    # Uncomment this to enable any form of FTP write command.
    #
    # 可写，允许用户上传文件
    write_enable=YES
    #
    # Default umask for local users is 077. You may wish to change this to 022,
    # if your users expect that (022 is used by most other ftpd's)
    #local_umask=022
    #
    # Uncomment this to allow the anonymous FTP user to upload files. This only
    # has an effect if the above global write enable is activated. Also, you will
    # obviously need to create a directory writable by the FTP user.
    #anon_upload_enable=YES
    #
    # Uncomment this if you want the anonymous FTP user to be able to create
    # new directories.
    #anon_mkdir_write_enable=YES
    #
    # Activate directory messages - messages given to remote users when they
    # go into a certain directory.
    dirmessage_enable=YES
    #
    # If enabled, vsftpd will display directory listings with the time
    # in  your  local  time  zone.  The default is to display GMT. The
    # times returned by the MDTM FTP command are also affected by this
    # option.
    use_localtime=YES
    #
    # Activate logging of uploads/downloads.
    xferlog_enable=YES
    #
    # Make sure PORT transfer connections originate from port 20 (ftp-data).
    connect_from_port_20=YES
    #
    # If you want, you can arrange for uploaded anonymous files to be owned by
    # a different user. Note! Using "root" for uploaded files is not
    # recommended!
    #chown_uploads=YES
    #chown_username=whoever
    #
    # You may override where the log file goes if you like. The default is shown
    # below.
    #xferlog_file=/var/log/vsftpd.log
    #
    # If you want, you can have your log file in standard ftpd xferlog format.
    # Note that the default log file location is /var/log/xferlog in this case.
    #xferlog_std_format=YES
    #
    # You may change the default value for timing out an idle session.
    #idle_session_timeout=600
    #
    # You may change the default value for timing out a data connection.
    #data_connection_timeout=120
    #
    # It is recommended that you define on your system a unique user which the
    # ftp server can use as a totally isolated and unprivileged user.
    #nopriv_user=ftpsecure
    #
    # Enable this and the server will recognise asynchronous ABOR requests. Not
    # recommended for security (the code is non-trivial). Not enabling it,
    # however, may confuse older FTP clients.
    #async_abor_enable=YES
    #
    # By default the server will pretend to allow ASCII mode but in fact ignore
    # the request. Turn on the below options to have the server actually do ASCII
    # mangling on files when in ASCII mode.
    # Beware that on some FTP servers, ASCII support allows a denial of service
    # attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
    # predicted this attack and has always been safe, reporting the size of the
    # raw file.
    # ASCII mangling is a horrible feature of the protocol.
    #ascii_upload_enable=YES
    #ascii_download_enable=YES
    #
    # You may fully customise the login banner string:
    #ftpd_banner=Welcome to blah FTP service.
    #
    # You may specify a file of disallowed anonymous e-mail addresses. Apparently
    # useful for combatting certain DoS attacks.
    #deny_email_enable=YES
    # (default follows)
    #banned_email_file=/etc/vsftpd.banned_emails
    #
    # You may restrict local users to their home directories.  See the FAQ for
    # the possible risks in this before using chroot_local_user or
    # chroot_list_enable below.
    #chroot_local_user=YES
    #
    # You may specify an explicit list of local users to chroot() to their home
    # directory. If chroot_local_user is YES, then this list becomes a list of
    # users to NOT chroot().
    # (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
    # the user does not have write access to the top level directory within the
    # chroot)

    # 防止ftp连接用户访问目录树之外的任何文件和命令 
    chroot_local_user=YES
    #chroot_list_enable=YES
    # (default follows)
    #chroot_list_file=/etc/vsftpd.chroot_list
    #
    # You may activate the "-R" option to the builtin ls. This is disabled by
    # default to avoid remote users being able to cause excessive I/O on large
    # sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
    # the presence of the "-R" option, so there is a strong case for enabling it.
    #ls_recurse_enable=YES
    #
    # Customization
    #
    # Some of vsftpd's settings don't fit the filesystem layout by
    # default.
    #
    # This option should be the name of a directory which is empty.  Also, the
    # directory should not be writable by the ftp user. This directory is used
    # as a secure chroot() jail at times vsftpd does not require filesystem
    # access.
    secure_chroot_dir=/var/run/vsftpd/empty
    #
    # This string is the name of the PAM service vsftpd will use.
    pam_service_name=vsftpd
    #
    # This option specifies the location of the RSA certificate to use for SSL
    # encrypted connections.
    rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
    rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
    ssl_enable=NO

    #  配置ftpuser01 ftp的根目录
    local_root =/home/ftpuser01/ftp

    # ftp被动模式ftp端口范围
    pasv_min_port = 40000
    pasv_max_port = 50000

    # Uncomment this to indicate that vsftpd use a utf8 filesystem.
    #utf8_filesystem=YES   

    // 这三项配置之后，只有在 /etc/vsftpd.userlist 配置的用户才能访问到
    userlist_enable=YES
    userlist_file=/etc/vsftpd.userlist
    userlist_deny=NO

tips：配置的有几个点需要注意

    // 灵活配置用户的ftp目录,不同用户进来就到不同的目录
    user_sub_token=$USER
    local_root=/home/$USER/ftp

    //userlist_deny 切换逻辑：当它设置为YES ，列表中的用户被拒绝FTP访问。 当它设置为NO ，只允许列表中的用户访问。

添加访问用户白名单到文件中

    echo "ftpuser01" | sudo tee -a /etc/vsftpd.userlist


重启服务

    sudo systemctl restart vsftpd


测试

    // 切换为自己主机的地址,以此输入用户名和密码
    ftp 127.0.0.1
    // 使用客服端连接不上时，检查配置，传输模式(主,被动)，字符集是否正确? 

常见ftp相关命令

    // 列出目录
    ls
    // 切换目录
    cd
    // 获取文件
    get /files/test.txt
    // 获取一批文件
    mget *.*
    // 上传文件
    put local.txt  remote.txt
    // 退出
    bye

#### rust编写客户端

* create io: https://crates.io/crates/ftp
* github : https://github.com/mattnenterprise/rust-ftp

简单示例


    extern crate ftp;
    use std::str;
    use std::io::Cursor;
    use ftp::FtpStream;

    fn main() {
     
        // 连接服务器
        let mut ftp_stream = FtpStream::connect("127.0.0.1:21").unwrap();
        // 登录
        let _ = ftp_stream.login("username", "password").unwrap();
        // 获取当前路径 
        println!("Current directory: {}", ftp_stream.pwd().unwrap());
        // 切换目录
        let _ = ftp_stream.cwd("files").unwrap();
        // get 下载文件
        let remote_file = ftp_stream.simple_retr("test.txt").unwrap();
        println!("Read file with contents\n{}\n", str::from_utf8(&remote_file.into_inner()).unwrap());

        let mut reader = Cursor::new("Hello from the Rust \"ftp\" crate!".as_bytes());
        // 上传文件
        let _ = ftp_stream.put("greeting.txt", &mut reader);
        println!("Successfully wrote greeting.txt");

        // Terminate the connection to the server.
        let _ = ftp_stream.quit();
    }

参考连接

* 搭建ftp:  https://www.howtoing.com/how-to-set-up-vsftpd-for-a-user-s-directory-on-ubuntu-18-04
* ftp命令:  http://imhuchao.com/323.html
* 配置相关: https://blog.csdn.net/aiynmimi/article/details/77012507




