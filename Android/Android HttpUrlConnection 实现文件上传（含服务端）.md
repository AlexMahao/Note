## Android HttpUrlConnection 实现文件上传（含服务端）

### 分析原理

首先实现文件上传肯定要通过Http Post 请求，因为Get 请求无法传输大文件。

使用Post请求传输文件，则Http协议中包含如下两点的改变：

- 请求头中定义表单请求的格式，传输的大小。
- 请求体中传输数据。

**请求头中定义表单请求的格式，传输的大小**

在请求头中，有两个参数`Content-Type`和`Content-Length`，分别定义传输的类型和传输的大小。

- `Content-Type`:`multipart/form-data; boundary=`+自定义的字符串
	
- `Content-Length`：请求体中的数据大小

**请求体中数据格式的定义**

```java 
------WebKitFormBoundaryT1HoybnYeFOGFlBR
Content-Disposition: form-data; name="user"

admin
------WebKitFormBoundaryT1HoybnYeFOGFlBR
Content-Disposition: form-data; name="file"; filename="ss.jpg"
Content-Type: image/jpeg

// 流的形式传输图片

------WebKitFormBoundaryT1HoybnYeFOGFlBR--

```

`WebKitFormBoundaryT1HoybnYeFOGFlBR`为自定义的字符串，也就是`Content-Type`中传入的`boundary`值

数据中以`------WebKitFormBoundaryT1HoybnYeFOGFlBR`作为分割，上面试传入了一个`user=admin`的普通参数，下面传输的是一个文件。


我们知道传输值时以键值对的形式存在，文件同样如此，`name=file`,`file`是键，`filename="ss.jpg"`，`ss.jpg`是值，代表了服务端获取到的文件名。`Content-Type: image/jpeg`，传输内容的类型，然后以字节的形式传输图片，最后用`------WebKitFormBoundaryT1HoybnYeFOGFlBR--`代表结束。

那么我们只需要按照这种格式构造传输内容即可

### Android 端代码实现

Android 端的实现，其实和java的实现几乎一致，唯一的区别就是图片地址的不同，在这里使用的是Java工程进行演示。

```java 
/**
 * 文件表单上传
 */
public class FileUpLoadTest {

	// 分割符
	private static final String BOUNDARY = "----WebKitFormBoundaryT1HoybnYeFOGFlBR";


	/**
	 * HttpUrlConnection　实现文件上传
	 * @param params 普通参数
	 * @param fileFormName 文件在表单中的键
	 * @param uploadFile 上传的文件
	 * @param newFileName 文件在表单中的值（服务端获取到的文件名）
	 * @param urlStr url
	 * @throws IOException
	 */
	public static void uploadForm(Map<String, String> params, String fileFormName, File uploadFile, String newFileName,
			String urlStr) throws IOException {

		if (newFileName == null || newFileName.trim().equals("")) {
			newFileName = uploadFile.getName();
		}
		StringBuilder sb = new StringBuilder();
		/**
		 * 普通的表单数据
		 */
		if (params != null) {
			for (String key : params.keySet()) {
				sb.append("--" + BOUNDARY + "\r\n");
				sb.append("Content-Disposition: form-data; name=\"" + key + "\"" + "\r\n");
				sb.append("\r\n");
				sb.append(params.get(key) + "\r\n");
			}
		}

		/**
		 * 上传文件的头
		 */
		sb.append("--" + BOUNDARY + "\r\n");
		sb.append("Content-Disposition: form-data; name=\"" + fileFormName + "\"; filename=\"" + newFileName + "\""
				+ "\r\n");
		sb.append("Content-Type: image/jpeg" + "\r\n");// 如果服务器端有文件类型的校验，必须明确指定ContentType
		sb.append("\r\n");

		byte[] headerInfo = sb.toString().getBytes("UTF-8");
		byte[] endInfo = ("\r\n--" + BOUNDARY + "--\r\n").getBytes("UTF-8");


		URL url = new URL(urlStr);
		HttpURLConnection conn = (HttpURLConnection) url.openConnection();
		conn.setRequestMethod("POST");
		// 设置传输内容的格式，以及长度
		conn.setRequestProperty("Content-Type", "multipart/form-data; boundary=" + BOUNDARY);
		conn.setRequestProperty("Content-Length",
				String.valueOf(headerInfo.length + uploadFile.length() + endInfo.length));
		conn.setDoOutput(true);

		OutputStream out = conn.getOutputStream();
		InputStream in = new FileInputStream(uploadFile);
		// 写入头部 （包含了普通的参数，以及文件的标示等）
		out.write(headerInfo);
		// 写入文件
		byte[] buf = new byte[1024];
		int len;
		while ((len = in.read(buf)) != -1) {
			out.write(buf, 0, len);
		}
		// 写入尾部
		out.write(endInfo);
		in.close();
		out.close();
		if (conn.getResponseCode() == 200) {
			System.out.println("文件上传成功");
		}
	}

	public static void main(String[] args) throws IOException {
		//需要上传的文件
		File file = new File("ss.png");
		// 普通参数
		HashMap<String , String> params = new HashMap<>();
		params.put("user", "admin");
		
		// conn上传
		uploadForm(params, "file", file, "ss.jpg", "http://localhost:8080/Web/UploadFile");

	}

}

```

### 服务端代码实现

服务端代码需要导入Apache 提供的两个jar包，分别是`commons-io-1.4.jar`和`commons-fileupload-1.2.1.jar`，[具体下载请点击](http://download.csdn.net/detail/fishfire/751772)


```java 
@WebServlet("/UploadFile")
public class UploadFile extends HttpServlet {
	public void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		
		response.setContentType("text/html");
		PrintWriter out = response.getWriter();
		
		
		// 创建文件项目工厂对象
		DiskFileItemFactory factory = new DiskFileItemFactory();

		// 设置文件上传之后保存到本地的路径
		String upload = this.getServletContext().getRealPath("/upload/");
		File file = new File(upload);
		
		if(!file.exists()){
			file.mkdirs();
		}
		System.out.println(upload);
		// 获取系统默认的临时文件保存路径，该路径为Tomcat根目录下的temp文件夹
		String temp = System.getProperty("java.io.tmpdir");
		// 设置缓冲区大小为 5M
		factory.setSizeThreshold(1024 * 1024 * 5);
		// 设置临时文件夹为temp
		factory.setRepository(new File(temp));
		// 用工厂实例化上传组件,ServletFileUpload 用来解析文件上传请求
		ServletFileUpload servletFileUpload = new ServletFileUpload(factory);

		// 解析结果放在List中
		try {
			List<FileItem> list = servletFileUpload.parseRequest(request);
		
			for (FileItem item : list) {
				// 获取字段标识，key
				String name = item.getFieldName();
				
				InputStream is = item.getInputStream();

				// 如果参数名是file  
				 if (name.contains("file")) {
					try {
						//将对应图片转化并保存
						inputStream2File(is,upload+"/"+item.getName());
					} catch (Exception e) {
						e.printStackTrace();
					}
				}else if(name.equals("user")){
					// 参数名是uesr，则代表是字符串，转化成字符串
					try {
						System.out.println(inputStream2String(is));
					} catch (Exception e) {
						e.printStackTrace();
					}
				}
			}

			out.write("success");
		} catch (FileUploadException e) {
			e.printStackTrace();
			out.write("failure");
		}

		out.flush();
		out.close();
	}

	// 流转化成字符串
	public static String inputStream2String(InputStream is) throws IOException {
		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		int i = -1;
		while ((i = is.read()) != -1) {
			baos.write(i);
		}
		return baos.toString();
	}

	// 流转化成文件
	public static void inputStream2File(InputStream is, String savePath) throws Exception {
		System.out.println("文件保存路径为:" + savePath);
		File file = new File(savePath);
		InputStream inputSteam = is;
		BufferedInputStream fis = new BufferedInputStream(inputSteam);
		FileOutputStream fos = new FileOutputStream(file);
		int f;
		while ((f = fis.read()) != -1) {
			fos.write(f);
		}
		fos.flush();
		fos.close();
		fis.close();
		inputSteam.close();

	}
}


```