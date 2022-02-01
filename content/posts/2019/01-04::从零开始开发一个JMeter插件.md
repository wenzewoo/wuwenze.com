+++
title = "从零开始开发一个JMeter插件"
date = "2019-01-04 06:39:00"
url = "archives/142"
tags = ["JMeter"]
categories = ["测试"]
+++

虽然JMeter自带的插件基本能满足大多数场景，但有时候也需要自定义一些插件来实现。网上的JMeter的插件开发文档稀少，通过本人的一些尝试，总结了一些JMeter插件开发相关的经验。

### JMeter的核心组件 ###

 *  `Timer` 定时器，用于配置每次sampling之间的等待时间。
 *  `Sampler` 取样器，如果是其他的协议需要实现其他协议的Sampler。
 *  `ConfigElement` 配置组件，主要用于定义前置配置。如数据库连接，csv输入数据集等。
 *  `Assertion` 断言，验证Sampler的结果是否符合预期。
 *  `PostProcessor` 后置处理器，一般用于对Sampler结果进行二次加工。
 *  `Visualizer` 将sampler的结果进行可视化展示。
 *  `Controller` 对sampler进行逻辑控制。
 *  `SampleListener` 监听器，一般用于保存sampler的结果等耗费时间的操作。

### JMeter插件加载机制 ###

通过阅读JMeter源码发现，它的加载插件机制是相当简单的，扫描扩展下的的所有实现了JMeterGUIComponent和TestBean接口的类，然后进行初始化。

```java
ClassFinder.findClassesThatExtend(
	JMeterUtils.getSearchPaths(), 
	new Class[] {JMeterGUIComponent.class, TestBean.class }
```

所以只要确保插件的jar包在扩展路径下即可，默认路径是: `JMETER_HOME/lib/ext`

### JMeter的GUI机制 ###

JMeter是基于Swing实现的，咱们直接继承JMeterGUIComponent接口的抽象实现类即可：

```bash
org.apache.jmeter.config.gui.AbstractConfigGui
org.apache.jmeter.assertions.gui.AbstractAssertionGui
org.apache.jmeter.control.gui.AbstractControllerGui
org.apache.jmeter.timers.gui.AbstractTimerGui
org.apache.jmeter.visualizers.gui.AbstractVisualizer
org.apache.jmeter.samplers.gui.AbstractSamplerGui
org.apache.jmeter.processor.gui.AbstractPostProcessorGui
...
```

![0c0109c2-05f4-4ca0-986c-36558df71547.png][]

### 例子 ###

本例子是一个后置处理器（CsvWriterPostProcessor），用于将取样器结果按照指定的格式写入CSV文件中。

#### 建立一个标准的Maven项目，其核心依赖如下： ####

```xml
<dependency>
  <groupId>org.apache.jmeter</groupId>
  <artifactId>ApacheJMeter_core</artifactId>
  <version>5.0</version>
</dependency>
<dependency>
  <groupId>org.apache.jmeter</groupId>
  <artifactId>ApacheJMeter_java</artifactId>
  <version>5.0</version>
</dependency>
<dependency>
  <groupId>net.sourceforge.javacsv</groupId>
  <artifactId>javacsv</artifactId>
  <version>2.0</version>
</dependency>
```

#### 实现AbstractPostProcessorGui，绘制界面： ####

```java
public class CsvWriterPostProcessorGui extends AbstractPostProcessorGui {
	public static final String WIKIPAGE = "CsvWriterPostProcessor";
	private JTextField filename, headers, columnVariables;
	private JCheckBox appendRecord;

	public CsvWriterPostProcessorGui() {
		super();
		this.initGui();
		this.initDefaultFields();
	}

	@Override
	public String getStaticLabel() {
		return JMeterPluginsUtils.prefixLabel("CsvWriter PostProcessor");
	}

	@Override
	public String getLabelResource() {
		return getClass().getCanonicalName();
	}

	@Override
	public void configure(TestElement element) {
		super.configure(element);
		if (element instanceof CsvWriterPostProcessor) {
			CsvWriterPostProcessor el = (CsvWriterPostProcessor) element;
			filename.setText(el.getFileName());
			headers.setText(el.getHeaders());
			columnVariables.setText(el.getColumnVariables());
			appendRecord.setSelected(el.isAppendRecord());
		}
	}

	@Override
	public TestElement createTestElement() {
		CsvWriterPostProcessor csvWriterPostProcessor = new CsvWriterPostProcessor();
		this.modifyTestElement(csvWriterPostProcessor);
		csvWriterPostProcessor.setComment(JMeterPluginsUtils.getWikiLinkText(WIKIPAGE));
		return csvWriterPostProcessor;
	}

	@Override
	public void modifyTestElement(TestElement element) {
		super.configureTestElement(element);
		if (element instanceof CsvWriterPostProcessor) {
			CsvWriterPostProcessor el = (CsvWriterPostProcessor) element;
			el.setFileName(filename.getText());
			el.setHeaders(headers.getText());
			el.setColumnVariables(columnVariables.getText());
			el.setAppendRecord(appendRecord.isSelected());
		}
	}

	@Override
	public void clearGui() {
		super.clearGui();
		this.initDefaultFields();
	}

	private void initGui() {
		setLayout(new BorderLayout(0, 5));
		setBorder(makeBorder());

		add(JMeterPluginsUtils.addHelpLinkToPanel(makeTitlePanel(), WIKIPAGE), BorderLayout.NORTH);

		JPanel mainPanel = new JPanel(new GridBagLayout());

		GridBagConstraints labelConstraints = new GridBagConstraints();
		labelConstraints.anchor = GridBagConstraints.FIRST_LINE_END;

		GridBagConstraints editConstraints = new GridBagConstraints();
		editConstraints.anchor = GridBagConstraints.FIRST_LINE_START;
		editConstraints.weightx = 1.0;
		editConstraints.fill = GridBagConstraints.HORIZONTAL;

		addToPanel(mainPanel, labelConstraints, 0, 1, new JLabel("FileName: ", JLabel.RIGHT));
		addToPanel(mainPanel, editConstraints, 1, 1, filename = new JTextField(20));
		JButton browseButton = new JButton("Browse...");
		addToPanel(mainPanel, labelConstraints, 2, 1, browseButton);
		GuiBuilderHelper.strechItemToComponent(filename, browseButton);
		browseButton.addActionListener(new BrowseAction(filename));

		addToPanel(mainPanel, labelConstraints, 0, 2, new JLabel("Headers: ", JLabel.RIGHT));
		addToPanel(mainPanel, editConstraints, 1, 2, headers = new JTextField(20));

		editConstraints.insets = new Insets(2, 0, 0, 0);
		labelConstraints.insets = new Insets(2, 0, 0, 0);
		addToPanel(mainPanel, labelConstraints, 0, 3, new JLabel("ColumnVariables: ", JLabel.RIGHT));
		addToPanel(mainPanel, editConstraints, 1, 3, columnVariables = new JTextField(20));

		addToPanel(mainPanel, labelConstraints, 0, 4, new JLabel("AppendRecord?: ", JLabel.RIGHT));
		addToPanel(mainPanel, editConstraints, 1, 4, appendRecord = new JCheckBox());

		JPanel container = new JPanel(new BorderLayout());
		container.add(mainPanel, BorderLayout.NORTH);
		add(container, BorderLayout.CENTER);
	}

	private void addToPanel(JPanel panel, GridBagConstraints constraints, int col, int row, JComponent component) {
		constraints.gridx = col;
		constraints.gridy = row;
		panel.add(component, constraints);
	}

	private void initDefaultFields() {
		filename.setText("email.token.csv");
		headers.setText("Email,Token");
		columnVariables.setText("email,token");
		appendRecord.setSelected(true);
	}
}
```

#### 实现PostProcessor，处理读取数据、写入CSV文件逻辑： ####

```java
public class CsvWriterPostProcessor extends AbstractTestElement
		implements PostProcessor {
	private static final Logger log = LoggingManager.getLoggerForClass();
	private static final String FILENAME = "CsvWriterPostProcessor.FileName";
	private static final String HEADERS = "CsvWriterPostProcessor.Headers";
	private static final String COLUMN_VARIABLES = "CsvWriterPostProcessor.ColumnVariables";
	private static final String APPEND_RECORD = "CsvWriterPostProcessor.AppendRecord";
	private static final String DEFAULT_CHARSET = "UTF-8";
	private static final char DEFAULT_CSV_SPLIT = ',';
	private static final String DEFAULT_CSV_COLUMN_VALUE = "-";


	@Override
	public void process() {
		this.doCsvWriter(this.getFileName(), this.getCsvHeaders(), this.getCsvColumns());
	}

	private String[] getCsvHeaders() {
		String headers = this.getHeaders();
		if (null == headers || headers.length() == 0) {
			return new String[0];
		}
		return headers.split(",");
	}

	private List<String[]> getCsvColumns() {
		List<String[]> csvColumns = new ArrayList<>();

		Integer maxMatchNr = -1;
		String columnVariableString = this.getColumnVariables();
		String[] columnVariables = (null != columnVariableString && columnVariableString.trim().length() != 0) ? columnVariableString.split(",") : new String[0];
		for (int i = 0; i < columnVariables.length; i++) {
			int _matchNr = this.getVariableAsInt(columnVariables[i] + "_matchNr", -1);
			if (_matchNr > maxMatchNr) {
				maxMatchNr = _matchNr;
			}
		}
		String[] firstColumns = new String[columnVariables.length];
		for (int i = 0; i < columnVariables.length; i++) {
			// get columnVariables
			firstColumns[i] = this.getVariableAsString(columnVariables[i], DEFAULT_CSV_COLUMN_VALUE);
		}
		if (!this.isEmptyColumns(firstColumns)) {
			csvColumns.add(firstColumns);
		}

		for (int i = 0; i < maxMatchNr; i++) {
			String[] bodyColumns = new String[columnVariables.length];
			for (int j = 0; j < columnVariables.length; j++) {
				// get columnVariables_matchNr
				bodyColumns[j] = this.getVariableAsString((columnVariables[j] + ("_" + (i + 1))), DEFAULT_CSV_COLUMN_VALUE);
			}
			if (!this.isEmptyColumns(bodyColumns)) {
				csvColumns.add(bodyColumns);
			}
		}
		return csvColumns;
	}


	private void doCsvWriter(String path, String[] csvHeader, List<String[]> csvColumns) {
		log.info("#0104 doCsvWriter path = " + path);

		if (null == csvColumns || csvColumns.size() == 0) {
			log.info("#0104 doCsvWriter error, csvColumns.size() == 0");
			return;
		}

		boolean isAppendRecord = this.isAppendRecord();
		if (isAppendRecord && csvHeader != null && csvHeader.length > 0) {
			CsvReader csvReader = null;
			try {
				csvReader = new CsvReader(path, DEFAULT_CSV_SPLIT, Charset.forName(DEFAULT_CHARSET));
				csvReader.readHeaders();

				String[] readerHeaders = csvReader.getHeaders();
				if (readerHeaders.length != csvHeader.length) {
					isAppendRecord = false;
				}

				for (int i = 0; i < readerHeaders.length; i++) {
					if (!readerHeaders[i].equals(csvHeader[i])) {
						isAppendRecord = false;
						break;
					}
				}
				if (readerHeaders.length > 0 && isAppendRecord) {
					csvHeader = null;
				}
			} catch (FileNotFoundException e) {
				// ignore
			} catch (IOException e) {
				e.printStackTrace();
			} finally {
				if (null != csvReader) {
					csvReader.close();
				}
			}
		}

		CsvWriter csvWriter = null;
		BufferedWriter bufferedWriter = null;
		try {
			bufferedWriter = new BufferedWriter(//
					new OutputStreamWriter(//
							new FileOutputStream(path, isAppendRecord), DEFAULT_CHARSET), 1024);
			csvWriter = new CsvWriter(bufferedWriter, DEFAULT_CSV_SPLIT);

			if (null != csvHeader && csvHeader.length > 0) {
				csvWriter.writeRecord(csvHeader);
				log.info("#0104 doCsvWriter writeRecord csvHeader = " + this.printArray(csvHeader));
			}
			for (String[] csvColumn : csvColumns) {
				csvWriter.writeRecord(csvColumn);
				log.info("#0104 doCsvWriter writeRecord csvColumn = " + this.printArray(csvColumn));
			}
			log.info("#0104 doCsvWriter success, csvColumns.size() == " + csvColumns.size());
		} catch (IOException e) {
			e.printStackTrace();
		} finally {
			if (null != csvWriter) {
				csvWriter.flush();
				csvWriter.close();
			}
			if (null != bufferedWriter) {
				try {
					bufferedWriter.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}
	}

	private boolean isEmptyColumns(String[] array) {
		if (null != array && array.length > 0) {
			int emptyCount = 0;
			for (int i = 0; i < array.length; i++) {
				if (null == array[i] || array[i].trim().length() == 0 || DEFAULT_CSV_COLUMN_VALUE.equals(array[i])) {
					emptyCount++;
				}
			}
			return emptyCount == array.length;
		}
		return true;
	}

	private String printArray(String[] array) {
		if (null != array && array.length > 0) {
			StringBuilder stringBuilder = new StringBuilder("[");
			for (String item : array) {
				stringBuilder.append(item).append(",");
			}
			if (stringBuilder.length() > 1) {
				stringBuilder.deleteCharAt(stringBuilder.length() - 1);
			}
			stringBuilder.append("]");
			return stringBuilder.toString();
		}
		return "[]";
	}

	public JMeterVariables getVars() {
		return this.getThreadContext().getVariables();
	}

	private String getVariableAsString(String key, String defaultVal) {
		Object value = this.getVars().getObject(key);
		if (null == value || (value instanceof String && ((String) value).trim().length() == 0)) {
			return defaultVal;
		}
		return String.valueOf(value);
	}

	private int getVariableAsInt(String key, int defaultVal) {
		Object value = this.getVars().getObject(key);
		if (null == value) {
			return defaultVal;
		}
		return Integer.parseInt(String.valueOf(value));
	}

	public void setFileName(String fileName) {
		this.setProperty(FILENAME, fileName);
	}

	public String getFileName() {
		return this.getPropertyAsString(FILENAME);
	}

	public void setHeaders(String headers) {
		this.setProperty(HEADERS, headers);
	}

	public String getHeaders() {
		return this.getPropertyAsString(HEADERS, DEFAULT_CSV_COLUMN_VALUE);
	}

	public void setColumnVariables(String columns) {
		this.setProperty(COLUMN_VARIABLES, columns);
	}

	public String getColumnVariables() {
		return this.getPropertyAsString(COLUMN_VARIABLES);
	}

	public void setAppendRecord(boolean appendRecord) {
		this.setProperty(APPEND_RECORD, appendRecord);
	}

	public boolean isAppendRecord() {
		return this.getPropertyAsBoolean(APPEND_RECORD, true);
	}
}
```

#### 打包并测试 ####

打包完成后，将jar放入`JMETER_HOME/lib/ext`目录中。

##### 添加取样器，请求相关接口，接口返回数据格式如下：

```json
{
	"result": {
		"_count": 100,
		"_total": 105,
		"_page": 1,
		"engineers": [{
			"user": {
				"email": "xxxxx@test.com",
				"id": 6008,
				"status": 1
			}
		},....]
	},
	"status": 0
}
```

##### 添加两个正则表达式提取器，将所有的Email、UserID取出。

```bash
// email = {"email":"(.*?)",
// userId = "id":(.*?),
```

![2019-08-19-073401][]

![2019-08-19-073402][]

##### 添加编写好的后置处理器（CsvWriterPostProcessor）

最终的结果如下
![abfab2ed-5959-451a-a3ee-a3a437532259.png][]
![4e11f0cd-bc77-4895-bb58-3e4c557f239e.png][]


[0c0109c2-05f4-4ca0-986c-36558df71547.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190104/0c0109c2-05f4-4ca0-986c-36558df71547.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073401]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190104/1a8b2b92-c478-4e8e-b170-1058d627655f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073402]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190104/bf066645-d959-4e75-b209-6d942d3cc957.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[abfab2ed-5959-451a-a3ee-a3a437532259.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190104/abfab2ed-5959-451a-a3ee-a3a437532259.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[4e11f0cd-bc77-4895-bb58-3e4c557f239e.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190104/4e11f0cd-bc77-4895-bb58-3e4c557f239e.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg