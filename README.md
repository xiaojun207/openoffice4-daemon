# Openoffice 4 headless daemon
Image running the OpenOffice 4 soffice daemon service, and solve the problem of Chinese code scrambling.(包括中文乱码问题)

This image was created following the https://github.com/rafaeltuelho/openoffice3-daemon. And thanks he

## Build this image:

```
 docker build --pull -t xiaojun207/openoffice4-daemon --build-arg OO_VERSION=4.1.7 .
```

## Run the container

```
docker run -d -u 123456 --name soffice -p 8100:8100 -v /data/:/data/ xiaojun207/openoffice4-daemon:latest
```

When you run this image the container will start the Openoffice daemon in headless mode listening on TCP port `8100` by default. To change this port pass the env var `SOFFICE_DAEMON_PORT`

>If you use java, you can use jodconverter connect the container, like this:

```
import java.io.File;
import java.io.FileNotFoundException;

import lombok.extern.slf4j.Slf4j;
import org.artofsolving.jodconverter.OfficeDocumentConverter;
import org.artofsolving.jodconverter.office.DefaultOfficeManagerConfiguration;
import org.artofsolving.jodconverter.office.OfficeManager;

...

	public static void main(String[] args){
    DefaultOfficeManagerConfiguration configuration = new DefaultOfficeManagerConfiguration();
    configuration.setOfficeHome("/opt/openoffice4");
    configuration.setPortNumbers(8100);
    configuration.setTaskExecutionTimeout(1000 * 60 * 5L);// 设置任务执行超时为5分钟
    configuration.setTaskQueueTimeout(1000 * 60 * 60 * 24L);// 设置任务队列超时为24小时
    OfficeManager officeManager = configuration.buildOfficeManager();
    String inputFile = "/data/a.docx";
    String pdfFile = "/data/a.pdf"; // 注意输出目录的docker写入权限，否则可能会失败
    System.out.println("进行文档转换转换:" + inputFile + " --> " + pdfFile + ",isRunning:" + officeManager.isRunning());
    OfficeDocumentConverter converter = new OfficeDocumentConverter(officeManager);
    converter.convert(new File(inputFile), new File(pdfFile));
	}

```
* Note the write permission of the output directory, otherwise it may fail.

## Verify the daemon port is listening for connections

```
docker exec -it soffice test
```

## Test soffice daemon using `unoconv`

The `unoconv` utility is available in this image! You can test a PDF conversion as follow:


 * first put some `.odt` or `.doc` files into a dir (eg: `~/pdfs`) in your host. 
 * then run the container attaching that dir as a Docker Volume and specifying the file you want to convert"

```
docker run \
 -v ~/pdfs:/pdfs:rw \
 xiaojun207/openoffice4-daemon \
 unoconv --connection 'socket,host=127.0.0.1,port=8100,tcpNoDelay=1;urp;StarOffice.ComponentContext' \
 -f pdf /pdfs/somefile.doc
```

 * now you should see the file converted to `.pdf` inside the dir mounted as Volume
  
## Add this container as sidecar for any app depends on Openoffice for any reason (eg. PDF generation).

 * import the image and create an Openshift `ImageStream`

```
oc import-image openoffice4-daemon --from=docker.io/xiaojun207/openoffice4-daemon --confirm --scheduled
```

 * edit your `DeploymentConfig` to include the `soffice` container inside your App **POD**

```yaml
...
    spec:
      containers:
        - image: >-
            docker.io/xiaojun207/openoffice4-daemon@sha256:<image tag sha256>
          imagePullPolicy: Always
          name: soffice
          ports:
            - containerPort: 8100
              protocol: TCP
...
  test: false
  triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
          - soffice
        from:
          kind: ImageStreamTag
          name: 'openoffice4-daemon:latest'
          namespace: demo-tomcat6
        lastTriggeredImage: >-
          docker.io/xiaojun207/openoffice4-daemon@sha256:<image tag sha256>
      type: ImageChange
...
```
