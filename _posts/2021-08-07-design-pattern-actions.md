---
layout: post
title: 设计模式实战
description: 
date: 2021-08-07
---

# 组合模式

组合模式（Composite Pattern），又叫部分整体模式，是用于把一组相似的对象当作一个单一的对象。

1. 意图：将对象组合成树形结构以表示"部分-整体"的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。
2. 主要解决：它在我们树型结构的问题中，模糊了简单元素和复杂元素的概念，客户程序可以像处理简单元素一样来处理复杂元素，从而使得客户程序与复杂元素的内部结构解耦。
3. 何时使用： 1、您想表示对象的部分-整体层次结构（树形结构）。 2、您希望用户忽略组合对象与单个对象的不同，用户将统一地使用组合结构中的所有对象。

## 答题卡制作工具

* 场景

答题卡的基本结构元件：定位块、标题、二维码、准考证号、缺考标记、单选题、解答题

<img src="../../../assets/images/design-pattern-actions/answersheet.png">

答题卡制作工具有两个核心任务：
1. 以html的形式来呈现整个答题卡的结构
2. 提供答题卡各结构元件相对于页面的精准位置信息

* 抽象

1. 答题卡中的每个结构元件都可以用一个div来呈现
2. 结构元件会有父子的层级关系
3. 使用css的绝对定位来布局各个结构元件
4. 每个结构元件负责自己的html生成，所有结构元件组织成父子的层级关系，通过递归层级调用的方式就可以生成用于呈现整个答题卡的html。这就是一种化繁为简的思想。
5. 每个结构元件都相对于父亲进行绝对定位，基于上述的父子层级关系，通过递归层级计算，就可以很方便的得到各个结构元件相对于页面的精准位置信息了。

* 类设计

``` typescript

// 所有结构元件的基类
export class Element {
  top?: number;
  left?: number;
  width?: number;
  height?: number;

  pTop?: number;
  pLeft?: number;

  parent?: any;
  children?: any = [];

  elementType?: string = '';

  constructor(top: number, left: number, width: number, height: number) {
    this.top = top;
    this.left = left;
    this.width = width;
    this.height = height;
  }

  writeHtmlBegin(answersheet: any): void {
  }

  protected writeHtml(answersheet: any): void {
  }

  writeHtmlEnd(answersheet: any): void {
  }

  calcLocation(): void {
  }
}

// 答题卡的一面纸张
export class PageBox extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'PageBox';
  }
}

// 定位点
export class LocatePoint extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'LocatePoint';
  }
}

// 选择题
export class ChoiceBox extends Element {
  constructor(top: number, left: number, width: number, height: number) {
    super(top, left, width, height);
    this.elementType = 'ChoiceBox';
  }
}

```

## 客观题填涂识别

* 场景

<img src="../../../assets/images/design-pattern-actions/object-rec.png">

1. 前提：已能准确获得客观题每一区块、每一个选项的图片。
2. 填涂识别的任务是判断每一个选项是否填涂。
3. 理论上只需要计算出每一个选项的平均灰度值和填涂面积，然后设置一个阈值，就可以判断出选项是否填涂。
4. 现实场景要复杂很多，例如填涂较淡、擦除、印刷问题等，根本就不是一个阈值能搞定。
5. 算法上考虑在一个题目范围内各个选项间进行对比，一个区块范围内相同选项间进行对比，整个答题卡范围内相同选项间进行对比，从而使填涂识别具有更强的适应性

* 抽象

1. 由于需要进行横向、纵向的对比分析，答题卡中的客观题可抽象成三个结构元件：客观题区块、客观题、选项
2. 三个结构元件组成父子的层级关系，相互关联，随时可访问
3. 如此组织代码，很容易实现题目范围内、区块范围内、整个答题卡范围内的对比

* 类设计

``` c++
namespace eiomr {

	class Option {
		
	public:
		Option(int idx);
		~Option();

	public:
		int idx;
		float fillArea;
		int meanValue;
		bool isSelected;

	};

	class Topic {

	public:
		Topic(int idx, int num,int optionCnt,int inIdx);
		~Topic();

	public:
		vector<Option*> options;
		int optionCnt;  
		int idx;
		int num;
		int inIdx;

	};

	class Branch {

	public:
		Branch(int topicCnt, vector<int> ixList, vector<int> numList, cv::Mat* asImg);
		~Branch();

	public:
		vector<Topic*> topics;
		vector<int> startXVec;
		vector<int> startYVec;
		int left;
		int top;
		int width;
		int height;
		cv::Mat* asImg;

	};

}
```

# 建造者模式

建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

一个Builder类会一步一步构造最终的对象。该Builder类是独立于其他对象的。

## 扫描上传时异常检测

* 场景

1. 一张答题卡扫描上传后，要对其存在的异常进行检测（例如：图像残缺、定位点异常、准考证号异常、客观题异常、选作题异常等）
2. 异常检测时需要结合很多方面的信息，具体如下：

    ``` txt
    基础信息（包含：考试、学科、是否手阅卡）
    配置信息（包含：是否检测客观题异常、主观题0分是否算作异常等配置）
    计划参加考试的学生信息
    答题卡模板信息（包含：是否系统卡、图片数量、是否双张、双张卡每一张所包含的试题序号等）
    试题信息（包含：客观题信息、选作题信息、主观题信息等）
    ```

3. 综合了各方面的信息后，就可以检测出一张答题卡所包含的各种异常了

* 抽象

1. 需要有一个异常检测工具，此工具能组合异常检测时所依赖的各种信息，能够检测各种异常
2. 需要有一个异常检测工具的生成器，能获取到各面的信息，全方位管理异常检测工具的生成过程

* 类设计

``` java
public class ScanDataProcessTool {

    /**
     * 当时考试是否是系统模板
     */
    private boolean isSystemTemplate = false;
    private boolean isHandMarking = false;
    /**
     * 模板图片数量
     */
    private Integer imgCnt;

    /**
     * 用户信息，
     */
    private LoginData loginData;
    private Map<String,String> configMap;


    private Exam exam;
    private ExamSession examSession;
    private List<Student> planStudents;
    private List<ExamTopic> topics;
    private List<ExamTopic> chooseTopics;
    private List<ExamTopic> objectTopics;
    private List<ExamTopic> subjectTopics;
    private Map<Integer, Integer> ixOpcnt;
    private List<String> ixNum;
    private Map<Integer,Set<Integer>> pageNumTopicIndexMap;
}
```

<img src="../../../assets/images/design-pattern-actions/ScanDataProcessToolBuilder.png">

# 代理模式

在代理模式（Proxy Pattern）中，一个类代表另一个类的功能。这种类型的设计模式属于结构型模式。

在代理模式中，我们创建具有现有对象的对象，以便向外界提供功能接口。

## 扫描上传及异常处理兼容双张

* 场景

1. 扫描上传及异常处理先前只支持单张答题卡，现需要能兼容双张答题卡
2. 双张答题卡与单张答题卡的处理逻辑差异很大，需要分开独立处理

* 抽象

1. 扫描上传与异常处理的业务逻辑主要是由SheetScanService、ScanExceptionService这两个Service来承接的
2. 所以可以创建这两个Service的代理类，在代理类中进行是否双张的判断，从而将单张与双张的逻辑处理独立分开。Controller层只需要调用代理类提供的统一接口，而不用关心单张与双张的差异细节。
3. 这样组织代码后，单张与双张的业务逻辑各自独立，互不影响，保持绝对的稳定性，后期业务需求的修改与维护也会非常轻松。

* 类设计

``` java
public class SheetScanServiceAgent {
    public static void uploadStudentAnswer(StudentAnswer studentAnswer, Integer imgCnt, Integer markingState){
        SheetScanService sheetScanService= SpringContextUtil.getBean(SheetScanService.class);
        boolean isMultiPage=ComUtil.isMultiPage(studentAnswer.getExamId(),studentAnswer.getSubjectId());
        if(isMultiPage){
            sheetScanService.uploadStudentAnswer_multipage(studentAnswer,imgCnt,markingState);
        }else{
            sheetScanService.uploadStudentAnswer(studentAnswer,imgCnt,markingState);
        }
    }

    public static void deleteSheets(Integer examId, Integer subjectId, List<String> uuIds){
        SheetScanService sheetScanService= SpringContextUtil.getBean(SheetScanService.class);
        boolean isMultiPage=ComUtil.isMultiPage(examId,subjectId);
        if(isMultiPage){
            sheetScanService.deleteSheets_multipage(examId,subjectId,uuIds);
        }else{
            sheetScanService.deleteSheets(examId,subjectId,uuIds);
        }
    }
}

public class ScanExceptionServiceAgent {

    public static Map<String,List<String>> checkRepeatAbnormal(Integer examId, Integer subjectId, List<Integer> stuIds,Integer pageNum){
        ScanExceptionService scanExceptionService= SpringContextUtil.getBean(ScanExceptionService.class);
        boolean isMultiPage=ComUtil.isMultiPage(examId,subjectId);
        if(isMultiPage){
            return scanExceptionService.checkRepeatAbnormal_multipage(examId, subjectId, stuIds,pageNum);
        }else{
            return scanExceptionService.checkRepeatAbnormal(examId,subjectId,stuIds);
        }
    }

    public static List<StudentScore> getSubjectScore(Integer examId, Integer subjectId, String sheetId) {
        ScanExceptionService scanExceptionService= SpringContextUtil.getBean(ScanExceptionService.class);
        boolean isMultiPage=ComUtil.isMultiPage(examId,subjectId);
        if(isMultiPage){
            return scanExceptionService.getSubjectScore_multipage(examId,subjectId,sheetId);
        }else{
            return scanExceptionService.getSubjectScore(examId,subjectId,sheetId);
        }
    }
}
```

# 责任链模式

责任链模式（Chain of Responsibility Pattern）为请求创建了一个接收者对象的链。

## word2html

* 场景

1. 需要将一份word格式的试卷转化为html格式，以便于进行划题及试题存储
2. word转html的整个过程是很复杂，其间有许多步骤需要处理，大致如下：

    ``` txt
    step1：下载word文档
    step2：将doc格式转化成docx格式
    step3：将Omath公式转化成latex
    step4：将MathType公式转化成latex
    step5：将word转化成html
    step6：对字符编码进行处理
    step7：html中域公式的处理
    step8：html中MathType公式的一些处理（填充img标签的LaTeX属性、将gif图片转换为高清png图片、调整图片的位置）
    step9：html中Omth公式的一些处理
    step10：上传html中所有的图片
    step11：上传html
    step12: 结束
    ```
3. 组织好这么多步骤的代码，使其结构清晰，便于分工协作，便于灵活插拔，是一项挑战也是一种艺术

* 抽象

1. 可以将需要转化的word试卷、最终的html格式、各处理步骤的中间结果，组织为一个对象，称之为转化上下文
2. 需要设计一个对象将各个处理步骤包含起来，对处理步骤的顺序、处理步骤的加入与删除进行统一管理，称之为管道
3. 每个处理步骤称之为处理节点，从转化上下文获取需要的信息，并进行相应的处理，处理后的结果再反馈给转化上下文，以便进行后续处理
4. 这样组织代码后，结构清晰，可以团队进行分工协作，处理步骤可灵活插拔，处理步骤职责单一，研发效率与质量得以保障

* 类设计

``` java
{% raw %}
/**
 * Word转换上下文。
 * <pre>
 * 1、一个管道会关联一个上下文。
 * 2、上下文用于在管道的各个处理器之间传递信息。
 * </pre>
 *
 * @author nse
 */
public class TranContext {

	private static String workPathRoot;

	static {
		workPathRoot = EtcdUtil.getV("/job/word-trans-job/workPathRoot", "");
//		workPathRoot="C:\\me\\temp\\word2html";
	}

	private String workPath;

	private String paperId;

	/**
	 * 试卷来源。
	 * <ul>
	 *     <li>1 新资源加工平台</li>
	 *     <li>2 原划题平台</li>
	 * </ul>
	 */
	private int source = 1;

	private String wordFileName;

	private String htmlFileName;

	private Document document;

	public TranContext(String paperId) {
		this.paperId = paperId;
		this.workPath = Paths.get(workPathRoot, paperId).toString();
	}

	public TranContext(String paperId, String workPathRoot, String wordFileName) {
		TranContext.workPathRoot = workPathRoot;
		this.paperId = paperId;
		this.workPath = Paths.get(workPathRoot, paperId).toString();
		this.wordFileName = wordFileName;
	}
}

public interface PipeLine {

	TranContext getContext();

	void start();

	BaseProc getNextProcessor(int curIndex);

}

/**
 * 默认管道。
 *
 * @author nse
 */
public class DefaultPipeLine implements PipeLine {

	private final Logger logger = LoggerFactory.getLogger(DefaultPipeLine.class);

	private final TranContext context;

	protected List<BaseProc> processors = new ArrayList<>();

	public DefaultPipeLine(TranContext context) {
		this.context = context;

		this.processors.add(new DownloadWordProc(this, 0));
		this.processors.add(new Doc2DocxProc(this, 1));
		this.processors.add(new Omath2TexProc(this, 2));
		this.processors.add(new MathType2TexProc(this, 3));
		this.processors.add(new Word2HtmlProc(this, 4));
		this.processors.add(new CharSetProc(this, 5));
		this.processors.add(new HtmlEqProc(this, 6));
		this.processors.add(new HtmlMathTypeProc(this, 7));
		this.processors.add(new HtmlOmathProc(this, 8));
		this.processors.add(new UploadImgProc(this, 9));
		this.processors.add(new UploadHtmlProc(this, 10));
		this.processors.add(new EndProc(this, 11));

	}

	@Override
	public TranContext getContext() {
		return context;
	}

	@Override
	public void start() {

		try {
			this.processors.get(0).process();
		} catch (Exception e) {
			logger.error(String.format("word2html转换失败！paperId：{%s}。", this.context.getPaperId()), e);
			String message = e.getMessage();
			if (message != null && message.length() > 10000) {
				message = message.substring(0, 10000);
			}
			if (this.context.getSource() == 1) {
				SpringContextUtil.getBean(ResPaperService.class).convertFail(this.context.getPaperId(), Machine.getInstance(), message);
			} else {
				SpringContextUtil.getBean(PaperService.class).convertFail(this.context.getPaperId(), message);
			}
		}
	}

	@Override
	public BaseProc getNextProcessor(int curIndex) {
		int nextIndex = curIndex + 1;
		if (nextIndex >= this.processors.size()) {
			return null;
		}
		return this.processors.get(nextIndex);
	}

}

/**
 * 处理器基类。
 *
 * @author nse
 */
public abstract class BaseProc {

	protected Logger logger = LoggerFactory.getLogger(BaseProc.class);

	protected static final String CHARSET = "UTF-8";

	private PipeLine pipeLine;

	private int index;

	public BaseProc(PipeLine pipeLine, int index) {
		this.pipeLine = pipeLine;
		this.index = index;
	}

	public void process() {
		this.exec();

		BaseProc proc = this.getPipeLine().getNextProcessor(this.getIndex());
		if (proc != null) {
			proc.process();
		}
	}

	protected abstract void exec();
}
{% endraw %}
```

<img src="../../../assets/images/design-pattern-actions/processor.png">

# 装饰器模式

装饰器模式（Decorator Pattern）允许向一个现有的对象添加新的功能，同时又不改变其结构。这种类型的设计模式属于结构型模式，它是作为现有的类的一个包装。

这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。

## 根据品牌动态展示菜单及跳转到对应的域名地址

* 场景

1. 现有的菜单展示，已经根据用户角色进行了权限区分
2. 系统新增品牌管理后，各个品牌都有各自要展示的菜单范围，但菜单的角色权限维持不变
3. 各品牌都有自己的域名，菜单跳转地址的域名需要对应变化

* 抽象

1. 在原有的菜单处理逻辑后面增加一点修饰处理，就可以满足需求
2. 根据品牌需要展示的菜单范围，对原有菜单列表进行过滤，即可返回品牌需要的菜单
3. 将菜单跳转地址的域名替换成对应品牌的域名

* 类设计

``` java
@RequestMapping("getNav")
@ResponseBody
public JsonResultWithData<List<NavItem>> getNav(HttpServletRequest request, HttpSession session, Company company) {
  CurUser curUser = CurUserUtil.get(request, session);

  if (null == curUser) {
    throw new CFBizException("用户未登录。");
  }

  User user = userService.get(Integer.parseInt(curUser.getUid()));
  if (null == user) {
    throw new CFBizException("未获取到用户信息。");
  }

  List<Role> roles = user.getRoles();

  // 培训机构添加个性化错题本导航
  if (curUser.isTrainingSchool()) {
    roles.add(Role.trainingSchoolAdmin);
  }

  if (CollectionUtils.isEmpty(roles)) {
    throw new CFBizException(String.format("用户[%s]角色信息异常。", user.getId()));
  }
  List<NavItem> navItems = navConfig.getNavByRole(user, roles, curUser);

  List<NavItem> result = Lists.newArrayList();
  for (NavItem item : navItems) {
    if (item.getNeed()) {
      result.add(item);
    }
  }

  List<NavItem> result1 = Lists.newArrayList();
  Map<String, NavItem> titleNavItems = navItems.stream().collect(Collectors.toMap(NavItem::getTitle, t -> t));
  List<Menu> menus = company.getMenus().stream().sorted(Comparator.comparing(Menu::getIndex)).collect(Collectors.toList());
  for (Menu menu : menus) {
    // 根据品牌的菜单范围保留菜单
    NavItem navItem = titleNavItems.get(menu.getTitle());
    if (null == navItem){
      continue;
    }

    // 根据品牌的域名替换菜单的跳转地址
    String url = navItem.getUrl().replace(eiduoRootUrl,"https://"+company.getDomain());
    navItem.setUrl(url);
    result1.add(navItem);
  }

  return new JsonResultWithData<List<NavItem>>(result1);
}
```

# 参考资料

[设计模式-菜鸟教程](https://www.runoob.com/design-pattern/design-pattern-tutorial.html)