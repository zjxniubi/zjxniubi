import tkinter as tk  # 导入tkinter模块用于创建GUI应用
from tkinter import filedialog  # 导入filedialog用于加载图片
from PIL import Image, ImageTk  # 导入PIL模块进行图像处理，并将其转换为Tkinter可处理格式
import cv2  # 导入OpenCV模块用于图像处理
import numpy as np  # 导入numpy模块进行矩阵运算


import logging  # 导入logging模块记录日志信息

logging.basicConfig(format='%(asctime)s:%(levelname)s:%(message)s', level=logging.INFO)

class App:
    # 初始化GUI(图形用户界面)
    def __init__(self, root):
        self.root = root

        # 定义图像和遮罩变量
        self.img = None  # 定义img变量，用于存储加载的图片
        self.mask = None  # 定义mask变量，用于存储图片的遮罩信息
        self.file_path = None  # 定义存储文件路径的变量
        self.need_refresh = False  # 定义是否需要刷新的标志
        self.current_method = None  # 定义当前选择的算法

        # 创建按钮和画布
        self.create_widgets()  # 调用创建按钮和画布的函数

        # 绑定画笔动作
        self.bind_brush_action()  # 调用绑定画笔动作的函数

        # 初始化其他变量
        self.prev_pts = []  # 存储前一笔画的点
        self.pen_strokes = []  # 存储全部笔画数据

    # 创建载入图片按钮
    def create_widgets(self):
        self.load_button = tk.Button(self.root, text="载入图片", command=self.load_image)  # 创建载入图片按钮，点击后调用load_image函数
        self.load_button.pack()  # 加载按钮到窗口

        # 创建画布框架
        self.image_frame = tk.Frame(self.root)  # 定义一个新的框架存储画布
        self.image_frame.pack()  # 加载新的框架到窗口

        # 创建显示图片的画布
        self.canvas_width = 350  # 定义画布宽度
        self.canvas_height = int(self.canvas_width*0.75)  # 定义画布高度，宽高比为4:3
        self.canvas1 = tk.Canvas(self.image_frame, width=self.canvas_width, height=self.canvas_height)  # 创建显示原图的画布
        self.canvas1.pack(side=tk.LEFT)  # 加载画布到框架
        self.canvas2 = tk.Canvas(self.image_frame, width=self.canvas_width, height=self.canvas_height)  # 创建显示处理后图像的画布
        self.canvas2.pack(side=tk.LEFT)  # 加载画布到框架

        # 创建操作按钮框架
        self.button_frame = tk.Frame(self.root)  # 定义一个新的框架存储操作按钮
        self.button_frame.pack()  # 加载新的框架到窗口

        # 创建FMM和NS按钮
        self.fmm_button = tk.Button(self.button_frame, text="FMM", command=self.apply_fmm)  # 创建FMM算法按钮，点击后调用apply_fmm函数
        self.fmm_button.pack(side=tk.LEFT)  # 加载按钮到框架
        self.ns_button = tk.Button(self.button_frame, text="NS", command=self.apply_ns)  # 创建NS算法按钮，点击后调用apply_ns函数
        self.ns_button.pack(side=tk.LEFT)  # 加载按钮到框架

        # 创建撤销按钮
        self.undo_button = tk.Button(self.button_frame, text="Undo", command=self.undo)  # 创建撤销按钮，点击后调用undo函数
        self.undo_button.pack(side=tk.LEFT)  # 加载按钮到框架

        # 创建刷新按钮
        self.refresh_button = tk.Button(self.button_frame, text="Refresh", command=self.refresh)  # 创建刷新按钮，点击后调用refresh函数
        self.refresh_button.pack(side=tk.LEFT)  # 加载按钮到框架

        # 创建滑动条
        self.slider = tk.Scale(self.root, from_=3, to=51, orient=tk.HORIZONTAL, length=200, resolution=2,
                            command=self.slider_update, label="窗口大小")  # 创建调整窗口大小的滑动条，滑动后调用slider_update函数
        self.slider.set(7)  # 设置滑动条默认值为7
        self.slider.pack()  # 加载滑动条到窗口

    # 绑定画笔动作
    def bind_brush_action(self):
        self.canvas1.bind("<B1-Motion>", self.draw_line)  # 绑定左键拖动事件，调用draw_line函数
        self.canvas1.bind("<ButtonRelease-1>", self.reset_prev_point)  # 绑定左键释放事件，调用reset_prev_point函数

    # 载入图像和遮罩, 确保图像的高度和宽度按照画布的尺寸进行调节
    def load_img_mask(self):
        logging.info(self.file_path)  # 输出文件路径到日志
        self.img = cv2.imread(self.file_path)  # 读取图像到img变量
        logging.info(self.img.shape)  # 输出图像尺寸到日志
        self.mask = np.zeros(self.img.shape[:2], np.uint8)  # 创建和图像大小相同的遮罩
        h, w = self.img.shape[:2]  # 获取图像高度和宽度

        # 缩放图像和遮罩
        # 计算 self.canvas_height/h 与 self.canvas_width/w 的最小值作为调节图像的比例
        ratio_height = self.canvas_height / h
        ratio_width = self.canvas_width / w
        ratio =min(ratio_height, ratio_width)  # TODO

        # 利用以上计算出的缩放最小比例 ratio 对 self.img 和 self.mask 进行缩放
        new_height = int(h * ratio)
        new_width = int(w * ratio)
        self.img =cv2.resize(self.img, (new_width, new_height))  # TODO  # 缩放图像
        self.mask = cv2.resize(self.mask, (new_width, new_height)) # TODO  # 缩放遮罩

    # 载入图像
    def load_image(self):
        if not self.need_refresh:
            self.file_path = filedialog.askopenfilename()  # 打开文件选择框选择图片
            logging.info("载入图像: %s", self.file_path.split('/')[-1])  # 输出选择的图片名称到日志
        else:
            self.need_refresh = False  # 设置刷新标志为False

        # 清空画布上的内容
        self.pen_strokes = []  # 清空笔画数据
        self.canvas2.delete("all")  # 清空处理后图像画布内容

        if self.file_path:
            self.load_img_mask()  # 调用载入图像和遮罩的函数

            # 将OpenCV图像转换为PIL图像
            pil_img = Image.fromarray(cv2.cvtColor(self.img, cv2.COLOR_BGR2RGB))  # 将OpenCV图像转换为PIL格式
            self.photo = ImageTk.PhotoImage(pil_img)  # 将PIL图像转换为Tkinter可处理格式
            self.canvas1.create_image(0, 0, anchor=tk.NW, image=self.photo)  # 显示图像到画布上

    # 保存笔画数据
    def reset_prev_point(self, event):
        self.pen_strokes.append(self.prev_pts)  # 将前一笔画数据存入笔画列表
        logging.info("已经保存了: %s 笔画", len(self.pen_strokes))  # 输出已保存的笔画数量到日志
        self.prev_pts = []  # 清空前一笔画数据

    # 画线
    def draw_line(self, event):
        pt = (event.x, event.y)  # 获取鼠标当前位置坐标
        if self.img is not None:
            if self.prev_pts:
                pen_width = 5  # 定义画笔宽度
                cv2.line(self.img, self.prev_pts[-1], pt, (255, 255, 255), pen_width)  # 在图像上画线
                cv2.line(self.mask, self.prev_pts[-1], pt, 255, pen_width)  # 在遮罩上画线
                pil_img = Image.fromarray(cv2.cvtColor(self.img, cv2.COLOR_BGR2RGB))  # 将OpenCV图像转换为PIL格式
                self.photo = ImageTk.PhotoImage(pil_img)  # 将PIL图像转换为Tkinter可处理格式
                self.canvas1.create_image(0, 0, anchor=tk.NW, image=self.photo)  # 显示图像到画布上
            self.prev_pts.append(pt)  # 将鼠标当前位置坐标加入到前一笔画数据列表中

    # 应用FMM算法
    def apply_fmm(self):
        self.current_method = 'fmm'  # 设置当前选择的算法为FMM
        if self.img is not None and self.mask is not None:
            # 从拖动条获取小窗大小 self.slider.get() 后对 self.img 进行 FMM 填充，掩模使用 self.mask
            slider_value = self.slider.get()
            res =self.fmm_implementation(self.img, self.mask, slider_value) # TODO  # 使用FMM算法修复图像
            pil_img = Image.fromarray(cv2.cvtColor(res, cv2.COLOR_BGR2RGB))  # 将OpenCV图像转换为PIL格式
            after_photo = ImageTk.PhotoImage(pil_img)  # 将PIL图像转换为Tkinter可处理格式
            self.canvas2.create_image(0, 0, anchor=tk.NW, image=after_photo)  # 显示处理后的图像到画布上
            self.canvas2.image = after_photo  # 存储处理后的图像
            logging.info("在图像上应用 FMM 算法")  # 输出日志信息到控制台

    # 应用NS算法
    def apply_ns(self):
        self.current_method = 'ns'  # 设置当前选择的算法为NS
        if self.img is not None and self.mask is not None:
            # 从拖动条获取小窗大小 self.slider.get() 后对 self.img 进行 NS 填充，掩模使用 self.mask
            res = self.ns_algorithm_implementation(self.img, self.mask, slider_value)  # TODO  # 使用NS算法修复图像
            pil_img = Image.fromarray(cv2.cvtColor(res, cv2.COLOR_BGR2RGB))  # 将OpenCV图像转换为PIL格式
            after_photo = ImageTk.PhotoImage(pil_img)  # 将PIL图像转换为Tkinter可处理格式
            self.canvas2.create_image(0, 0, anchor=tk.NW, image=after_photo)  # 显示处理后的图像到画布上
            self.canvas2.image = after_photo  # 存储处理后的图像
            logging.info("在图像上应用 NS 算法")  # 输出日志信息到控制台


    # 刷新画布
    def refresh(self):
        self.need_refresh = True  # 设置刷新标志为True
        self.current_method = None  # 清空当前选择的算法
        self.load_image()  # 调用载入图像的函数

    # 撤销操作
    def undo(self):
        logging.info("撤销笔画")  # 输出撤销信息到日志
        if self.pen_strokes:  # 如果笔画列表不为空
            self.pen_strokes.pop()  # 移除最后一个笔画数据

            logging.info("剩余笔画数: %s", len(self.pen_strokes))  # 输出剩余笔画数量到日志
            self.load_img_mask()  # 调用载入图像和遮罩的函数

            for prev_pts in self.pen_strokes:  # 遍历笔画列表
                for i in range(len(prev_pts) - 1):  # 遍历笔画点
                    cv2.line(self.img, prev_pts[i], prev_pts[i + 1], (255, 255, 255), 8)  # 在图像上画线
                    cv2.line(self.mask, prev_pts[i], prev_pts[i + 1], 255, 8)  # 在遮罩上画线

            pil_img = Image.fromarray(cv2.cvtColor(self.img, cv2.COLOR_BGR2RGB))  # 将OpenCV图像转换为PIL格式
            self.photo = ImageTk.PhotoImage(pil_img)  # 将PIL图像转换为Tkinter可处理格式
            self.canvas1.create_image(0, 0, anchor=tk.NW, image=self.photo)  # 显示图像到画布上

    # 滑动条值更新回调函数
    def slider_update(self, val):
        if self.current_method == 'fmm':  # 如果当前选择的算法为FMM
            self.apply_fmm()  # 调用应用FMM算法的函数
        elif self.current_method == 'ns':  # 如果当前选择的算法为NS
            self.apply_ns()  # 调用应用NS算法的函数
        else:
            pass  # 当前没有选择算法，不进行操作

root = tk.Tk()  # 创建Tkinter窗口对象
app = App(root)  # 创建App类实例
root.mainloop()  # 启动Tkinter窗口消息循环

