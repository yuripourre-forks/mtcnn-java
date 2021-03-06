# MTCNN Face Detection for Java, using Tensorflow and ND4J

[ ![Download](https://api.bintray.com/packages/big-data/maven/mtcnn-java/images/download.svg) ](https://bintray.com/big-data/maven/mtcnn-java/_latestVersion)

> Note: This is still Work In Progress!

`Java` and `Tensorflow` implementation of the [MTCNN Face Detector](https://arxiv.org/abs/1604.02878). Based on David Sandberg's [FaceNet's MTCNN](https://github.com/davidsandberg/facenet/tree/master/src/align) 
`python` implementation and the original [Zhang, K et al. (2016) ZHANG2016](https://arxiv.org/abs/1604.02878) paper and [Matlab implementation](https://github.com/kpzhang93/MTCNN_face_detection_alignment).

[<img align="right" src="https://raw.githubusercontent.com/tzolov/mtcnn-java/master/src/test/resources/docs/scdf-face-detection-2.gif" alt="scdf-mtcnn" hspace="10" vspace="10"></img>](https://github.com/tzolov/computer-vision/blob/master/spring-cloud-starter-stream-processor-face-detection-mtcnn/README.adoc) It reuses the `PNet`, `RNet` and `ONet` Tensorflow models build in [FaceNet's MTCNN](https://github.com/davidsandberg/facenet/tree/master/src/align) and 
initialized with the original [weights](https://github.com/kpzhang93/MTCNN_face_detection_alignment/tree/master/code/codes/MTCNNv2/model). [Here](https://github.com/davidsandberg/facenet/pull/866) 
you can find how to freeze the TF models.

> Note that the required Tensorflow models are already pre-bundled with this project! No need to download or freeze those by yourself.

The MTCNN technique involves significant amount of linear algebra operations, such as multi-dimensional array computations and so. 
Therefore the [ND4J](https://deeplearning4j.org/docs/latest/nd4j-overview) library is leveraged for implementing all (pre)processing steps 
required for flowing the data through the `PNet`, `RNet` and `ONet` Tensorflow networks. Furthermore [JavaCV](https://github.com/bytedeco/javacv) is 
leveraged for image manipulation and the `ND4J-Tensorflow` GraphRunner is used to inferring the pre-trained tensorflow models. 
The combination of those libraries allows to exchange processing states between the `ND4J`, `JavaCV` and `Tensorflow` with little data churn. 
It also provides off-heap memory management and native support for GPU and BLAS processor features.         

> NOTE: You can find the original NumPy/Python code snippets as inline comments in front of the equivalent ND4J java implementations.

## Quick Start

The [FaceDetectionSample1.java](./src/test/java/net/tzolov/cv/mtcnn/sample/FaceDetectionSample1.java) demonstrates how to use `MtcnnService` for detecting faces in images.
![Input Image](./src/test/resources/docs/AnnotatedImage.png)

Here is the essence this sample:

```java
// 1. Create face detection service.
MtcnnService mtcnnService = new MtcnnService(30, 0.709, new double[] { 0.6, 0.7, 0.7 });

try (InputStream imageInputStream = new DefaultResourceLoader() .getResource("classpath:/pivotal-ipo-nyse.jpg").getInputStream()) {
    // 2. Load the input image (you can use http:/, file:/ or classpath:/ URIs to resolve the input image
    BufferedImage inputImage = ImageIO.read(imageInputStream);
    // 3. Run face detection
    FaceAnnotation[] faceAnnotations = mtcnnService.faceDetection(inputImage);
    // 4. Augment the input image with the detected faces
    BufferedImage annotatedImage = MtcnnUtil.drawFaceAnnotations(inputImage, faceAnnotations);
    // 5. Store face-annotated image
    ImageIO.write(annotatedImage, "png", new File("./AnnotatedImage.png"));
    // 6. Print the face annotations as JSON
    System.out.println("Face Annotations (JSON): " + new ObjectMapper().writeValueAsString(faceAnnotations));
}
```
It takes an input image detect the faces, produces json annotations and augments the image with the faces. 

The face annotation json format looks like this:

```json
[ {
  "bbox" : { "x" : 331, "y" : 92, "w" : 58, "h" : 71 }, "confidence" : 0.9999871253967285,
  "landmarks" : [ {
    "type" : "LEFT_EYE", "position" : { "x" : 346, "y" : 120 } }, {
    "type" : "RIGHT_EYE", "position" : { "x" : 374, "y" : 119 } }, {
    "type" : "NOSE", "position" : { "x" : 359, "y" : 133 } }, {
    "type" : "MOUTH_LEFT", "position" : { "x" : 347, "y" : 147 } }, {
    "type" : "MOUTH_RIGHT", "position" : { "x" : 371, "y" : 147 },
  } ]
}, { ... 
```
## Rea-Time Face Detection with Spring Cloud Data Flow 
The [spring-cloud-starter-stream-processor-face-detection-mtcnn](https://github.com/tzolov/computer-vision/tree/master/spring-cloud-starter-stream-processor-face-detection-mtcnn) is 
Spring Cloud Data Flow processor that allows detecting and faces in real time from input image or video streams.

## Maven setup

Use the following dependency to add the `mtcnn` utility to your project 
```xml
<dependency>
  <groupId>net.tzolov.cv</groupId>
  <artifactId>mtcnn</artifactId>
  <version>0.0.4</version>
</dependency>
```
Also register `jcentral` to your list of maven repository (it is available out of the box for Gradle).
```xml
<repositories>
    <repository>
      <id>jcenter</id>
      <url>https://jcenter.bintray.com/</url>
    </repository>
</repositories>
```

## Benchmarking

Use this [MtcnnServiceBenchmark](https://github.com/tzolov/mtcnn-java/blob/master/src/test/java/net/tzolov/cv/mtcnn/beanchmark/MtcnnServiceBenchmark.java) to perform some basic benchmarking. You can change the image URI to test
the performance with different images.   

## Caveats

The ND4J, DataVec, ND4J-Tensorflow and JavaCV all rely on core libraries written in C++ (for performance reasons) and wrapped with JNI (e.g JavaCPP) wrapper. 
This approach has many great advantages such as support for GPU and BLAS math features, off-heap data management, low latency minimal data churn. 
 Still the approach apparently can induce significant jar footprint. By default if you bundle support for OS platforms (e.g. linux, android, windows, macos ..) 
 the target Spring Boot jar can grow to the insane 1GB jar footprint!

If you know what your target platform is going to be then you can remedy this situation by setting the `-Djavacpp.platform=` property. For example `-Djavacpp.platform=macosx-x86_64` for MacOS target platform.

Another possible solution might be to try to substitute the ND4J, DataVec and JavaCV stack using the newly released 
[]Tensorflow Java Ops API (ver. 1.10+)](https://github.com/tensorflow/tensorflow/releases/tag/v1.10.0) later appear to have the superset of what above stack 
provides but in a single C++ library with much small footprint. 
  
     
