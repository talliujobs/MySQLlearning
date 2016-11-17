使用MySQL函数插入测试数据信息
----
# 生成32位随机字符串
	select replace(uuid(), '-', '');

# 生成0-100的随机数值
	select CEIL((RAND() * 100));

# 函数声明语法

	delimiter //
	create procedure myProc() 
	begin
	declare id INT;
	set id=1;
	while id < 10 do
	insert into students values(id, REPLACE(uuid(), "-", ""),FLOOR(RAND() * 100));
	set id=id+1;
	end while;
	end //

# 函数调用语法
	call myProc();

# 删除函数
	drop procedure myProc;

# 初始化表
	TRUNCATE students;

# 添加用户
	insert into user(host,user,password) values('127.0.0.1','wbl',PASSWORD('123456'));

# 添加用户并赋予权限
	GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP
	ON 数据库.数据表
	TO '用户名'@'IP地址'
	IDENTIFIED BY '密码';

# 刷新权限
	FLUSH PRIVILEGES;
