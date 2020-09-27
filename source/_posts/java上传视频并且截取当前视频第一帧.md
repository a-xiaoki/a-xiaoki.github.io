---
title: java上传视频并且截取当前视频第一帧
date: 2020-09-27 21:53:02
tags: Java
categories: Java
---
java实现上传视频文件并且截取视频第一帧，保存到数据库。

这是我通过查找其他相关资料结合自己的想法写的一个有关于移动端拍摄上传视频，并且截取视频第一帧的方法，个人觉得比调用ffffmpeg要简单方便很多。

注释都写的很清楚的，就看你们的造化了。

maven依赖第一个好像是没有用到的。至于是真是假，自己去验证。

### maven相关：
``` bash
    <!--截取视频第一帧-->
    <dependency>
      <groupId>org.bytedeco</groupId>
      <artifactId>javacpp-presets</artifactId>
      <version>1.4.3</version>
    </dependency>

    <dependency>
      <groupId>org.openpnp</groupId>
      <artifactId>opencv</artifactId>
      <version>3.4.2-1</version>
    </dependency>
    <dependency>
      <groupId>org.bytedeco</groupId>
      <artifactId>javacpp</artifactId>
      <version>1.4.3</version>
    </dependency>
    <dependency>
      <groupId>org.bytedeco</groupId>
      <artifactId>javacv-platform</artifactId>
      <version>1.4.3</version>
    </dependency>
```
### 控制层：Controller
```bash
    /***
     * 获取指定视频的帧并保存为图片至指定目录
     * @param request
     * @param files
     * @return
     * @throws IOException
     */
    @RequestMapping(value = "/userUpload",method = RequestMethod.POST)
    @ResponseBody
    public ResultDate userUpload(@RequestParam(value = "files", required = false) MultipartFile files,
                                 HttpServletRequest request,
                                 @RequestParam(value = "title") String title) throws Exception {

        try {
            KidsUserVideo userVideo =new KidsUserVideo();
            //填补内容
            userVideo.setUserVideoTitle(title);
            userVideo.setUserVideoUptime(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date(System.currentTimeMillis())));//设置上传时间
            //获取文件名称
            String fileName = files.getOriginalFilename();

            // 自定义方式产生文件名
            String serialName = String.valueOf(System.currentTimeMillis());
            // 截取文件格式
            String fileType = fileName.substring(fileName.lastIndexOf("."));

            // 上传视频存放路径
            String filePath = request.getSession().getServletContext().getRealPath("/temp");
            String realPath = request.getSession().getServletContext().getRealPath(File.separator);
            System.out.println(realPath);

            //开始上传
            File f = new File(filePath);
            if (!f.exists()) {//如果没有这个目录，
                f.mkdirs();//就创建一个新的目录
            }
            if (files.getSize() > 100 * 1024 * 1024) {//判断上传的视频文件是否过大
                return ResultDate.build(403,"上传失败！您上传的文件太大,系统允许最大文件500M");
            }
            //上传视频
            //上传视频存放的地址
            String videoPath = filePath + "/videos/" + serialName + fileType;
            //截取图片存放的地址
            String imgPath = filePath + "/images/";
            files.transferTo(new File(videoPath)); // 转存文件

            File img = new File(imgPath);
            if (!img.exists()) {//如果没有这个目录，
                img.mkdirs();//就创建一个新的目录
            }
            //调用截取视频第一帧的方法
            boolean falg = mediaService.executeCodecs(videoPath,imgPath,serialName);
            //System.out.println("返回结果："+falg);

            //获取ip地址
            String IP = Inet4Address.getLocalHost().toString();
            int index = IP.indexOf('/');
            String ip = IP.substring(index+1, IP.length());
            String localhostIp = "http://" + ip + ":8080/";
            
            userVideo.setUserVideoPath(localhostIp+"temp/videos/"+serialName + fileType);
            userVideo.setUserVideoPicture(localhostIp+"temp/images/" + serialName + ".jpg");
            if(falg==true){//判断返回结果
                //等于true的话，就上传截图成功
                return ResultDate.ok(mediaService.saveMedia(userVideo));
            }
            return ResultDate.build(403,"上传截图异常！请重试。");
        } catch (Exception e) {
            logger.error("上传视频异常：", e);
        }
        //其他错误
        return ResultDate.build(403,"上传错误！");
    }
```
### 业务层：ServiceImpl
```bash
    /**
     * 获取指定视频的帧并保存为图片至指定目录
     * @param filePath 视频存放的地址
     * @param targerFilePath 截图存放的地址
     * @param targetFileName 截图保存的文件名称
     * @return
     * @throws Exception
     */
	@Override
	public boolean executeCodecs(String filePath, String targerFilePath, String targetFileName) throws Exception {
		try{
			FFmpegFrameGrabber ff = FFmpegFrameGrabber.createDefault(filePath);
			ff.start();
			String rotate =ff.getVideoMetadata("rotate");
			Frame f;
			int i = 0;
			while (i <1) {
				f =ff.grabImage();
				IplImage src = null;
				if(null !=rotate &&rotate.length() > 1) {
					OpenCVFrameConverter.ToIplImage converter =new OpenCVFrameConverter.ToIplImage();
					src =converter.convert(f);
					f =converter.convert(rotate(src, Integer.valueOf(rotate)));
				}
				doExecuteFrame(f,targerFilePath,targetFileName);
				i++;
			}
			ff.stop();
			return true;
		}catch (Exception e){
			e.printStackTrace();
		}
		return false;
	}

	/*
	 * 旋转角度的
	 */
	public static IplImage rotate(IplImage src, int angle) {
		IplImage img = IplImage.create(src.height(), src.width(), src.depth(), src.nChannels());
		opencv_core.cvTranspose(src, img);
		opencv_core.cvFlip(img, img, angle);
		return img;
	}

	public static void doExecuteFrame(Frame f, String targerFilePath, String targetFileName) {

		if (null ==f ||null ==f.image) {
			return;
		}
		Java2DFrameConverter converter =new Java2DFrameConverter();
		String imageMat ="jpg";
		String FileName =targerFilePath + File.separator +targetFileName +"." +imageMat;
		BufferedImage bi =converter.getBufferedImage(f);
		System.out.println("width:" + bi.getWidth());//打印宽、高
		System.out.println("height:" + bi.getHeight());
		File output =new File(FileName);
		try {
			ImageIO.write(bi,imageMat,output);
		}catch (IOException e) {
			e.printStackTrace();
		}
	}
```
代码有部分比较冗余，可以根据自己需求删减。

此方法已验证。可以完美的截取到想要的视频帧。
只不过我这里做的只是将其照片存放在服务器本地，如果有单独的图片服务器，可以将存放地址换一下。

另外在CSDN上都有发布：[java上传视频并且截取当前视频第一帧](https://blog.csdn.net/qq_39248900/article/details/84789068)

欢迎评论，指出不足~~~~~~
