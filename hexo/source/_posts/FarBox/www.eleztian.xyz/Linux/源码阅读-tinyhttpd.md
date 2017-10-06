---
date: 2016-10-10 22:07
title: '源码阅读-tinyhttpd'
categories: "Linux"
tags: [Linux, c]
keyworlds: 嵌入式, Linux, c, tinyhttpd
description:
---
>tinyhttpd 是一个不到 500 行的超轻量型 Http Server很适合初学者，帮助我们快速掌握unix socket编程 和 http请求流程。看完所有源码，真的感觉有很大收获，无论是 unix 的编程，还是 GET/POST 的 Web 处理流程，都清晰了不少。


#直接从[github下载](https://github.com/eleztian/tinyhttpd_read.git)


#注释源码如下：
```c
#include <stdio.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <ctype.h>
#include <strings.h>
#include <string.h>
#include <sys/stat.h>
#include <pthread.h>
#include <sys/wait.h>
#include <stdlib.h>
#define ISspace(x) isspace((int)(x))
#define SERVER_STRING "Server: jdbhttpd/0.1.0\r\n"
// 从套接字监听HTTP请求
void accept_request(int);
// 返回错误请求代码
void bad_request(int);
// 读取服务器上的文件到套接字
void cat(int, FILE *);
// 执行cgi发生错误
void cannot_execute(int);
// 吧错误信息写到perror并退出
void error_die(const char *);
// 执行cgi
void execute_cgi(int, const char *, const char *, const char *);
// 读取一行套接字
int get_line(int, char *, int);
// 把http响应头写到套接字
void headers(int, const char *);
// 找不到文件
void not_found(int);
// 调用cat把吴福气文件返回客户端
void serve_file(int, const char *);
// 初始化http server
int startup(u_short *);
// 返回给浏览器表明收到的 HTTP 请求所用的 method 不被支持
void unimplemented(int);
/**********************************************************************/
/* A request has caused a call to accept() on the server port to
 * return.  Process the request appropriately.
 * Parameters: the socket connected to the client */
/**********************************************************************/
void accept_request(int client)
{
	char buf[1024];
	int numchars;
	char method[255];
	char url[255];
	char path[512];
	size_t i, j;
	struct stat st;
	int cgi = 0;      /* becomes true if server decides this is a CGI
	                   * program */
	char *query_string = NULL;
	/* 读取一行数据 */
	numchars = get_line(client, buf, sizeof(buf));
	printf("buf:%s\n", buf);
	i = 0; j = 0;
	/* 将数据 复制到method中，直到空格或长度超过 */
	while (!ISspace(buf[j]) && (i < sizeof(method) - 1))
	{
		method[i] = buf[j];
		i++; j++;
	}
	method[i] = '\0';
	printf("method:%s\n", method);
	/* 如果不是GET请求 或 POST请求 */
	if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
	{
		/* 返回请求类型不支持到客户端 */
		printf("uniplemented");
		unimplemented(client);
		return;
	}
	/*如果是POST请求 则执行cgi */
	if (strcasecmp(method, "POST") == 0)
		cgi = 1;

	i = 0;
	/* 转到下一个空格后的数据*/
	/* 跳过空格  */	
	while (ISspace(buf[j]) && (j < sizeof(buf)))
		j++;
	/* 获取url */
	while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
	{
		url[i] = buf[j];
		i++; j++;
	}
	url[i] = '\0';
	if (strcasecmp(method, "GET") == 0)
	{
		query_string = url;
		/* 获取url参数 */
		while ((*query_string != '?') && (*query_string != '\0'))
			query_string++;
		/* 如果有参数 ，执行cgi*/
		if (*query_string == '?')
		{
			cgi = 1;
			/* 跳过？*/
			*query_string = '\0';
			query_string++;
		}
	}
	/* 生成文件地址path */
	sprintf(path, "htdocs%s", url);
	printf("%s", path);
	/* 如果url为 / 这访问 index.html*/
	if (path[strlen(path) - 1] == '/')
		strcat(path, "index.html");
	/* 打开path文件*/
	if (stat(path, &st) == -1) {
		/* 打开文件失败 */
		while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
			numchars = get_line(client, buf, sizeof(buf));
		/* 返回403 错误 */
		not_found(client);
	}
	else /* 打开文件成功 */
	{
		if ((st.st_mode & S_IFMT) == S_IFDIR)
			strcat(path, "/index.html");
		if ((st.st_mode & S_IXUSR) ||
		  	(st.st_mode & S_IXGRP) ||
		  	(st.st_mode & S_IXOTH)    )
			cgi = 1;
		if (!cgi) /* 发送文件内容到客户端 */
			serve_file(client, path);
		else /* 执行cgi */
			execute_cgi(client, path, method, query_string);
		}

	close(client);
}

/**********************************************************************/
/* Inform the client that a request it has made has a problem.
 * Parameters: client socket */
/**********************************************************************/
void bad_request(int client)
{
	char buf[1024];

	sprintf(buf, "HTTP/1.0 400 BAD REQUEST\r\n");
	send(client, buf, sizeof(buf), 0);
	sprintf(buf, "Content-type: text/html\r\n");
	send(client, buf, sizeof(buf), 0);
	sprintf(buf, "\r\n");
	send(client, buf, sizeof(buf), 0);
	sprintf(buf, "<P>Your browser sent a bad request, ");
	send(client, buf, sizeof(buf), 0);
	sprintf(buf, "such as a POST without a Content-Length.\r\n");
	send(client, buf, sizeof(buf), 0);
}

/**********************************************************************/
/* Put the entire contents of a file out on a socket.  This function
 * is named after the UNIX "cat" command, because it might have been
 * easier just to do something like pipe, fork, and exec("cat").
 * Parameters: the client socket descriptor
 *             FILE pointer for the file to cat */
/**********************************************************************/
void cat(int client, FILE *resource)
{
	char buf[1024];
	/* 读取文件内容到buf */
	fgets(buf, sizeof(buf), resource);
	/* 读取玩所有内容并发送到客户端 */
	while (!feof(resource))
	{
		send(client, buf, strlen(buf), 0);
		fgets(buf, sizeof(buf), resource);
	}
}

/**********************************************************************/
/* Inform the client that a CGI script could not be executed.
 * Parameter: the client socket descriptor. */
/**********************************************************************/
void cannot_execute(int client)
{
	char buf[1024];

	sprintf(buf, "HTTP/1.0 500 Internal Server Error\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "Content-type: text/html\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "<P>Error prohibited CGI execution.\r\n");
	send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Print out an error message with perror() (for system errors; based
 * on value of errno, which indicates system call errors) and exit the
 * program indicating an error. */
/**********************************************************************/
void error_die(const char *sc)
{
	/* 打印错误信息 */
	perror(sc);
	exit(1);
}

/**********************************************************************/
/*CGI(Common Gateway Interface 通用网关接口)
 * 作用：
 *	提取用户提交的表单信息
 *	处理表单信息
 *	输出，返回html响应*/

/* Execute a CGI script.  Will need to set environment variables as
 * appropriate.
 * Parameters: client socket descriptor
 *             path to the CGI script */
/**********************************************************************/
void execute_cgi(int client, const char *path,
                 const char *method, const char *query_string)
{
	char buf[1024];
	int cgi_output[2];
	int cgi_input[2];
	pid_t pid;
	int status;
	int i;
	char c;
	int numchars = 1;
	int content_length = -1;
	
	buf[0] = 'A'; buf[1] = '\0';
	if (strcasecmp(method, "GET") == 0)
		while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
			numchars = get_line(client, buf, sizeof(buf));
	else    /* POST */
	{
		numchars = get_line(client, buf, sizeof(buf));
		while ((numchars > 0) && strcmp("\n", buf))
		{
			buf[15] = '\0';
			if (strcasecmp(buf, "Content-Length:") == 0)
				content_length = atoi(&(buf[16]));
			numchars = get_line(client, buf, sizeof(buf));
		}
		if (content_length == -1) 
		{
			bad_request(client);
			return;
		}
	}	
	
	sprintf(buf, "HTTP/1.0 200 OK\r\n");
	send(client, buf, strlen(buf), 0);
	
	if (pipe(cgi_output) < 0) 
	{
		cannot_execute(client);
		return;
	}
	if (pipe(cgi_input) < 0) 
	{
		cannot_execute(client);
		return;
	}
	
	if ( (pid = fork()) < 0 ) 
	{
		cannot_execute(client);
		return;
	}
	if (pid == 0)  /* child: CGI script */
	{
		char meth_env[255];
		char query_env[255];
		char length_env[255];
			
		dup2(cgi_output[1], 1);
		dup2(cgi_input[0], 0);
		close(cgi_output[0]);
		close(cgi_input[1]);
		sprintf(meth_env, "REQUEST_METHOD=%s", method);
		putenv(meth_env);
		if (strcasecmp(method, "GET") == 0) 
		{
			sprintf(query_env, "QUERY_STRING=%s", query_string);
			putenv(query_env);
		}
		else {   /* POST */
			sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
			putenv(length_env);
		}
		execl(path, path, NULL);
		exit(0);
	} else {    /* parent */
		close(cgi_output[1]);
		close(cgi_input[0]);
		if (strcasecmp(method, "POST") == 0)
			for (i = 0; i < content_length; i++) 
			{
				recv(client, &c, 1, 0);
				write(cgi_input[1], &c, 1);
			}
		while (read(cgi_output[0], &c, 1) > 0)
		send(client, &c, 1, 0);
				
		close(cgi_output[0]);
		close(cgi_input[1]);
		waitpid(pid, &status, 0);
	}
}

/**********************************************************************/
/* Get a line from a socket, whether the line ends in a newline,
 * carriage return, or a CRLF combination.  Terminates the string read
 * with a null character.  If no newline indicator is found before the
 * end of the buffer, the string is terminated with a null.  If any of
 * the above three line terminators is read, the last character of the
 * string will be a linefeed and the string will be terminated with a
 * null character.
 * Parameters: the socket descriptor
 *             the buffer to save the data in
 *             the size of the buffer
 * Returns: the number of bytes stored (excluding null) */
/**********************************************************************/
int get_line(int sock, char *buf, int size)
{
	int i = 0;
	char c = '\0';
	int n;
	
	/* 循环读取直到长度超过size或者读到了行末 */
	while ((i < size - 1) && (c != '\n'))
	{
		/* 获取客户端发送的数据 （获取 1 个字符存放在c中)，
		 * flag = 0 从tcp buffer 中复制数据并将它从tcp buffer,
		 * 中移除*/
		n = recv(sock, &c, 1, 0);
		/* DEBUG printf("%02X\n", c); */
		if (n > 0)/* 读取到了数据 */
		{
			if (c == '\r')/*如果是行末 读取下一行 */
			{
				/* msg_peek 仅把tcp buffer中的数据督导buf中，
				 * 并不会把读取的数据从tcpbuffer中移除，再次
				 * 调用时任然读取到的时这个数据 */
				n = recv(sock, &c, 1, MSG_PEEK);
				/* DEBUG printf("%02X\n", c); */
				if ((n > 0) && (c == '\n'))
				    /* 如果接收到的是 \r\n则将\n从tcp buffer中移除 */
					recv(sock, &c, 1, 0);
				else
					c = '\n';
			}
			buf[i] = c;
			i++;
		}
		else
			c = '\n';
	}
	buf[i] = '\0';
	
	/* 返回读取到的个数 */
	return(i);
}

/**********************************************************************/
/* Return the informational HTTP headers about a file. */
/* Parameters: the socket to print the headers on
 *             the name of the file */
/**********************************************************************/
void headers(int client, const char *filename)
{
	char buf[1024];
	(void)filename;  /* could use filename to determine file type */

	strcpy(buf, "HTTP/1.0 200 OK\r\n");
	send(client, buf, strlen(buf), 0);
	strcpy(buf, SERVER_STRING);
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "Content-Type: text/html\r\n");
	send(client, buf, strlen(buf), 0);
	strcpy(buf, "\r\n");
	send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Give a client a 404 not found status message. */
/**********************************************************************/
void not_found(int client)
{
 char buf[1024];

 sprintf(buf, "HTTP/1.0 404 NOT FOUND\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, SERVER_STRING);
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "Content-Type: text/html\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "<HTML><TITLE>Not Found</TITLE>\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "<BODY><P>The server could not fulfill\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "your request because the resource specified\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "is unavailable or nonexistent.\r\n");
 send(client, buf, strlen(buf), 0);
 sprintf(buf, "</BODY></HTML>\r\n");
 send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Send a regular file to the client.  Use headers, and report
 * errors to client if they occur.
 * Parameters: a pointer to a file structure produced from the socket
 *              file descriptor
 *             the name of the file to serve */
/**********************************************************************/
void serve_file(int client, const char *filename)
{
	FILE *resource = NULL;
	int numchars = 1;
	char buf[1024];

	buf[0] = 'A'; buf[1] = '\0';
	while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
		numchars = get_line(client, buf, sizeof(buf));

	resource = fopen(filename, "r");
	if (resource == NULL)
		not_found(client);
	else
	{
		/* 向客户端发送 header和内容*/
		headers(client, filename);
		cat(client, resource);
	}
	fclose(resource);
}

/**********************************************************************/
/* This function starts the process of listening for web connections
 * on a specified port.  If the port is 0, then dynamically allocate a
 * port and modify the original port variable to reflect the actual
 * port.
 * Parameters: pointer to variable containing the port to connect on
 * Returns: the socket */
/**********************************************************************/
int startup(u_short *port)
{
	int httpd = 0; /* 文件描述符*/
	struct sockaddr_in name;

	httpd = socket(PF_INET, SOCK_STREAM, 0);
	/* 出错处理 */
	if (httpd == -1)
		error_die("socket");
	/* 清0 */
	memset(&name, 0, sizeof(name));
	name.sin_family = AF_INET;
	name.sin_port = htons(*port);
	name.sin_addr.s_addr = htonl(INADDR_ANY); /* 全网 */
	
	if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0)
		error_die("bind");
	
	if (*port == 0)  /* 如果端口为0 则申请一个临时端口 */
	{
		int namelen = sizeof(name);
		if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
		error_die("getsockname");
		*port = ntohs(name.sin_port);
	}

	if (listen(httpd, 5) < 0)
		error_die("listen");
	return(httpd);
}

/**********************************************************************/
/* Inform the client that the requested web method has not been
 * implemented.
 * Parameter: the client socket */
/**********************************************************************/
void unimplemented(int client)
{
	char buf[1024];

	sprintf(buf, "HTTP/1.0 501 Method Not Implemented\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, SERVER_STRING);
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "Content-Type: text/html\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "<HTML><HEAD><TITLE>Method Not Implemented\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "</TITLE></HEAD>\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "<BODY><P>HTTP request method not supported.\r\n");
	send(client, buf, strlen(buf), 0);
	sprintf(buf, "</BODY></HTML>\r\n");
	send(client, buf, strlen(buf), 0);
}
/**********************************************************************/
int main(int argc, char argv[])
{
	int server_sock = -1;
	u_short port = 0;
	int client_sock = -1;
	struct sockaddr_in client_name;
	int client_name_len = sizeof(client_name);
	pthread_t newthread;
	server_sock = startup(&port);
	printf("httpd running on port %d\n", port);
	while (1)
	{
		/* 开始阻塞接受 */
		client_sock = accept(server_sock,
		                   (struct sockaddr *)&client_name,
		                   &client_name_len);
		if (client_sock == -1)
			error_die("accept");
		/* 收到客户端连接，创建一个线程处理连接 */
		if (pthread_create(&newthread , NULL, accept_request, client_sock) != 0)
			perror("pthread_create");
	}
	close(server_sock);
	return(0);
}
```