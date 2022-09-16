# OPENCV项目（完整版）

```cpp
#include<opencv2/opencv.hpp>
#include<iostream>

using namespace cv;
using namespace std;

int cc = 0;
vector<Point> centers;
int center_x;
int center_y;

void fangkuai(Mat &imag);
void redpoint(Mat &image);
int main(int argc, char**argv) {
	Mat src = imread("C:\\Users\\26083\\Desktop\\机器视觉项目\\新\\测试图片\\H-30-25.jpg");
	Mat result;
	int flag1 = 0;
	src.copyTo(result);
	if (src.empty()) {
		cout << "can't find this photo!" << endl;
	}
	
	//若图像太大。则缩小10倍
	if (src.rows > 2000 || src.cols > 1000) {
		resize(src, src, Size(src.cols / 10, src.rows / 10), 0, 0, INTER_AREA);
		flag1 = 1;
	}
	imshow("src", src);
	fangkuai(src);
	redpoint(src);
	//判断原图像是否缩小，相应的改变所求坐标的大小
	if(flag1==1){
		circle(result, Point(centers[cc].x*10, centers[cc].y*10), 50, Scalar(0, 0, 255), 10, 8);
		circle(result, Point(center_x*10, center_y*10),50, Scalar(0, 0, 255), 10);
		printf("红点相当于方格纸中心的坐标:(%d,%d)", (center_x * 10 - centers[cc].x * 10), (centers[cc].y * 10 - center_y * 10));
	}
	if (flag1 ==0) {
		circle(result, Point(centers[cc].x , centers[cc].y ), 20, Scalar(0, 0, 255), 2, 8);
		circle(result, Point(center_x , center_y ), 26, Scalar(0, 0, 255), 2);
		printf("红点相当于方格纸中心的坐标:(%d,%d)", (center_x  - centers[cc].x), (centers[cc].y  - center_y ));
	}
	namedWindow("result", WINDOW_FREERATIO);
	imshow("result", result);
	
	waitKey(0);
	destroyAllWindows();
	return 0;
}

void fangkuai(Mat &imag) {
	//图片二值化
	imshow("src", imag);
	Mat gray, binary, open;
	cvtColor(imag, gray, COLOR_BGR2GRAY);
	threshold(gray, binary, 0, 255, THRESH_BINARY_INV | THRESH_OTSU);
	imshow("binary", binary);
	
	//开运算去噪声
	Mat k1 = getStructuringElement(MORPH_RECT, Size(3,3), Point(-1, -1));
	morphologyEx(binary, open, MORPH_OPEN, k1, Point(-1, -1));
	imshow("open", open);
	
	//闭运算消除孔洞
	Mat dilate1;
	Mat k2 = getStructuringElement(MORPH_RECT, Size(15,15), Point(-1, -1));
	morphologyEx(open, dilate1, MORPH_CLOSE, k2, Point(-1, -1));
	imshow("dilate", dilate1);
	dilate1 = 255 - dilate1;
	
	//寻找所有轮廓
	vector<vector<Point>>contours;
	vector<Vec4i>hierarchy;
	findContours(dilate1, contours, hierarchy, RETR_TREE, CHAIN_APPROX_SIMPLE);
	vector<Point> contour_max;
	Point contour_max_center;
	for (int i = 0; i < contours.size(); i++) {
		if (contourArea(contours[i])>500) {
			contour_max = contours[i];
		}
	}
	
	//寻找所有轮廓的中心
	
	centers.resize(contours.size());
	Moments M;
	for (int t=0; t < contours.size(); t++) {
		M = moments(contours[t]);
		centers[t] = Point(M.m10 / M.m00, M.m01 / M.m00);
	}
	M = moments(contour_max);
	contour_max_center = Point(M.m10 / M.m00, M.m01 / M.m00);
	
	//用距离最大轮廓最小的点做为中心点
	double distance = 0;
	double min = 0;
	
	for (int count = 0; count < contours.size(); count++) {
		distance = powf((centers[count].x - contour_max_center.x), 2) + powf((centers[count].y - contour_max_center.y), 2);
		if (distance == 0)continue;
		if (distance < min||min==0) {
			min = distance;
			cc = count;
		}
	}
}

void redpoint(Mat &image) {
	
	//将图像变换到HSV，分离出图像中红色的部分
	Mat imgHSV;
	Mat ce1, ce2, ce3;
	cvtColor(image, imgHSV, COLOR_BGR2HSV);
	inRange(imgHSV, Scalar(156, 43, 46), Scalar(180, 255, 255), ce1); //红色
	inRange(imgHSV, Scalar(0, 43, 46), Scalar(3, 255, 255), ce2); //红色
	add(ce1, ce2, ce3);
	imshow("red", ce3);
	
	//利用开运算和轮廓寻找函数分离出红点的最外层轮廓
	vector<vector<Point>>contours2;
	vector<Vec4i>hierarchy;
	for (int t = 2; contours2.size() != 1; t++) {
		Mat k5 = getStructuringElement(MORPH_ELLIPSE, Size(t, t));
		morphologyEx(ce3, ce3, MORPH_OPEN, k5);
		findContours(ce3, contours2, hierarchy, RETR_EXTERNAL, CHAIN_APPROX_SIMPLE);
		//drawContours(image, contours2, 0, Scalar(0, 0, 255), 2, 8);
	}
	imshow("ce3", ce3);

	//利用二阶矩计算轮廓的中心位置
	Moments M1;
	M1 = moments(contours2[0]);
	center_x = M1.m10 / M1.m00;
	center_y = M1.m01 / M1.m00;
}
```