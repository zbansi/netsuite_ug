# SuiteScritp 2.0开发实例 自定义工单+领料单 单据流转 打印

## 1. 业务流程

-------

```flow
st=>start: 开始
e=>end: 结束
op1=>operation: 创建工单
cond1=>condition: 是否已批准？
op2=>operation: 创建领料单
cond2=>condition: 是否已批准？
op3=>operation: 打印领料单

st->op1->cond1->op2->cond2->op3->e
cond1(yes)->op2
cond1(no)->op1
cond2(yes)->op3
cond2(no)->op2
```
## 2. 系统实现
### 2.1 自定义工单
	借助SuiteBuilder
### 2.2 自定义领料单
	借助SuiteBuilder
### 2.3 自定义审批工作流
	借助SuiteWorkflow
### 2.4 SuiteScript开发

------------
~~~mermaid
sequenceDiagram
工单 ->> 领料单: 你好！领料员xxx，我已被批准，请及时创建领料单!
工单 ->> 领料单: 创建领料单
工单 ->>领料单: 我有哪些领料单
领料单-->> 工单: 我的领料单号是xxx !
领料单->>领料单打印: 你好！我已被批准，可以打印出来啦！
领料单->> 领料单打印: 生成PDF
领料单打印-->>领料单: 打印日志：状态、次数
Note right of 领料单打印: 批量打印.
~~~
#### 2.4.1 工单状态变更提醒（需求）
- 功能设计：如果新增已批准工单，邮件、NS站内信通知领料员
- 技术设计：创建已保存搜索，并基于它设置邮件、NS remind提醒相应领料员

#### 2.4.2 从工单创建领料单（需求）
- **功能设计：**
		a) if 工单状态 = approved , 工单界面添加【生成领料单】按钮
	b) 点击此按钮，弹出领料单新增页面
	c) 领料单字段（来源工单、领料员、领料车间、仓库等）初始化赋值取自工单
- **技术设计：**
		a) UserEventScript  beforeLoad入口点函数完成页面按钮加载（参见附：WorkOrderToMaterialApplyBs.js）
	b) 按钮函数，url 模块API resolveRecord 解析记录类型，生成新url并跳转（参见附：CreateMAOBs.js）
	c) UserEventScript beforeLoad完成页面初始化，调用record模块 API 获取工单数据
	​	WorkOrderToMaterialApplyBs.js
	​	
	​	
#### 2.4.3 工单可以查询到关联领料单号（需求）
- **功能设计：**
	a) 工单添加一个子标签，下面可以查出领料单列表（头信息）
	b) 通过工单行，可以查看该行对应哪些领料单（行信息）
- **技术设计：**
  领料单创建时，需要记录来源单据（工单）的头行信息
  工单页面通过UserEventScript beforeLoad加载领料单信息。
#### 2.4.4 领料单批准后通知打印统计员（需求）
- **功能设计：**

  如果新增已批准领料单，邮件、NS站内信通知领料员

- **技术设计：**

  创建已保存搜索，并基于它设置邮件、NS remind提醒相应领料员

#### 2.4.5 已批准的领料单打印PDF（需求）

- **功能设计：**
	a) if 领料单审批状态 = approved， 添加 【打印领料单】 按钮
	b) 点击【打印领料单】按钮，出来PDF预览页面
	
- **技术设计：**
	a) UserEventScript API beforeLoad	完成页面按钮加载（参见MAOCreateFromWO.js）
```javascript
// 伪代码
   if (type == VIEW && status == 'APPROVED'){
    		 	addButton('打印领料单',' print_function')
    		 }
```
b) 打印按钮方法调用url模块resolveScript（用此API解析Suitelet）生成，并添加参数领料单id。Suitelet将领料单数据转换成xml，再调用render模块API(xmlToPdf)转换成PDF输出。(参见BSPrintMAO.js 和 BsPrintMAOSuitelet.js）
​	

```javascript
var u = url.resolveScript({
					scriptId : 'customscript_yq_bs_printmao_suitelet',
					deploymentId : 'customdeploy_yq_bs_printmao_suitelet',
					returnExternalUrl : false			
				});
```

#### 2.4.6 批量打印领料申请（需求）

- **功能设计：**
  a) 客制新页面，包括打量批印按钮、筛选字段、领料申请列表

  b) 筛选字段（审批状态）更换时，领料申请列表更新
  c) 点击【批量打印领料单】按钮，出来PDF预览页面

- **技术设计：**

  a) Suitelet输出页面

  b) Suitelet加载客户端脚本，通过fieldChanged API监听Suitelet页面筛选字段，解析生成并打开新url

  c) 打印按钮方法调用url模块resolveScript（用此API解析Suitelet）生成，并添加参数领料单id。

## A 附

### A.1 工单下推领料单源码

#### WorkOrderToMaterialApplyBs.js

```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType UserEventScript
 * @NModuleScope SameAccount
 * @author YQ12681
 */
////////////////Function Description//////////////////
// #1 工单页面加载前
// #1.1 如果是新建模式(type=CREATE), 什么也不做
// #1.2 如果是查看模式(type=VIEW),判断工单状态status
//		if status = 'APPROVED', 页面添加'生成领料单'按钮
//////////////////////////////////////////////////////
define([], function() {
	function beforeLoad(context) {
		// context. newRecord,type,form,request		
		if (context.type === context.UserEventType.CREATE)
			;
		/*context.newRecord.setValue({
			fieldId : 'custrecord_yq_workorder_status_bs',
			value : 1
		});

		* CREATE type下不能用getText，需要先setValue或setText
		* 否则会报错： SuiteScript 2.0 Error > SSS_INVALID_API_USAGE error
		* 
		* Cause: In Standard Mode, SSS_INVALID_API_USAGE error appears when a
		* user event scripts instantiates records by using the newRecord object
		* provided by the script context in the following scenaros: -- When the
		* script executes on a record that is being created, and the script
		* attempts to use Record.getText(options) without first using
		* Record.setText(options) for the same field. -- When the script
		* executes on an existing record or on a record being created through
		* copying, and the script uses Record.setValue(options) on a field
		* before using Record.getText(options) for the same field
		*/
		else if (context.type === context.UserEventType.VIEW) {
			var status = context.newRecord.getText({
				fieldId : 'custrecord_yq_workorder_status_bs'
			});

			// 如果工单状态已审批加载<生成领料单>按钮
			try {
				if (status == '已批准' || status =='APPROVED') {

					context.form.clientScriptModulePath = './CreateMAOBs.js';
					context.form.addButton({
						id : 'custpage_generate_mao',
						label : '生成领料单',
						functionName : 'create_mao_url'
					});
				}
				log.debug({
					title : 'Success',
					details : '添加-生成领料单-按钮成功！ status = ' + status
				});
			} catch (ex) {
				log.debug({
					title : ex.name,
					details : ex.message
				});
			}
		}
	}

	return {
		beforeLoad : beforeLoad
	};
});
```

#### CreateMAOBs.js
```javascript
/**
 * @NApiVersion 2.x
 * @author YQ12681
 */
////////////////Description//////////////
//
// 定义【生成领料单】按钮的方法
// 此方法生成并打开一个新URL
//
////////////////////////////////////////
define([ 'N/url', 'N/currentRecord' ], function(url, currentRecord) {
	return ({
		create_mao_url : function() {
			try {
				var u = url.resolveRecord({
					recordType : 'customrecord_yq_material_apply_bansi',
					recordId : '',
					isEditMode : true
				});
				log.debug({
					title : 'Success',
					details : 'url解析成功'
				});
			} catch (e1) {
				log.debug({
					title : e1.name,
					details : e1.message
				});
			}
			try {
				var record = currentRecord.get();
				var woid = record.id;
				/*var status = record.getValue({
					fieldId : 'custrecord_yq_workorder_status_bs'
				});*/
				log.debug({
					title : 'Success',
					details : '获取记录id =' + woid //+ ' status = ' + status
				});
			} catch (e2) {
				log.debug({
					title : e2.name,
					details : e2.message
				});
			}
			try {
				//if (status == 'APPROVED')
					window.location.href = u + '&woid=' + woid;
				log.debug({
					title : 'Success',
					details : 'url创建成功'
				});
			} catch (e3) {
				log.debug({
					title : e3.name,
					details : e3.message
				});
			}
		}
	});
});
```
####  MAOCreateFromWO.js
```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType UserEventScript
 * @author YQ12681 Bansi
 */
////////////////Function Description//////////////////
// #1 领料单页面加载前
// #1.1 如果是新建模式(type=CREATE),判断是否从工单自动创建
//		if true, 用工单数据初始化赋值领料单
//		if false, 手工填写赋值
// #1.2 如果是查看模式(type=VIEW),判断工单状态status
//		if status = 'APPROVED', 页面添加'打印领料单'按钮
//////////////////////////////////////////////////////
define(
		[ 'N/record' ],
		function(record) {
			function beforeLoad(context) {
				var maoRecord = context.newRecord;
				var type = context.type;
				var form = context.form;

				if (type == context.UserEventType.CREATE) {
					try {
						var woid = context.request.parameters.woid;
						if (woid) {
							var rec = record.load({
								type : 'customrecord_yq_workorder_bs',
								id : woid
							});
							//初始化赋值master
							var warehouse = rec.getValue({
								fieldId : 'custrecord_yq_workorder_shop_bs'
							});
							var clerk = rec.getValue({
								fieldId : 'custrecord_yq_workorder_lly_bs'
							});
							var workshop = rec.getValue({
								fieldId : 'custrecord_yq_workorder_dept_bs'
							});
							maoRecord.setValue(
									'custrecord_yq_mao_header_sourcewo_bs',
									woid);
							maoRecord.setValue(
									'custrecord_yq_mao_source_workshop_bs',
									workshop);
							maoRecord.setValue(
									'custrecord_yq_material_apply_requestor',
									clerk);
							maoRecord.setValue('custrecord_yq_warehouse_bansi',
									warehouse);
							maoRecord.setValue(
									'custrecord_yq_material_apply_note_bansi',
									'创建自工单:' + rec.getText({
										fieldId : 'name'
									}));

							//初始化赋值明细行
							//sublistid为recmach+子记录中作为外键到父记录的字段ID
							var sublistIdToSet = 'recmachcustrecord_yq_material_apply_detail_h_bs';
							var sublistIdToGet = 'recmachcustrecord_yq_workorder_detail_headid_bs';
							var count = rec.getLineCount(sublistIdToGet);
							for (var i = 0; i < count; i++) {
								maoRecord
										.setSublistValue(
												sublistIdToSet,
												'custrecord1',
												i,
												rec
														.getSublistValue(
																sublistIdToGet,
																'custrecord_yq_workorder_detail_item_bs',
																i));
								maoRecord
										.setSublistValue(
												sublistIdToSet,
												'custrecord_yq_material_apply_line_qyt_bs',
												i,
												rec
														.getSublistValue(
																sublistIdToGet,
																'custrecord_yq_workorder_detail_qty_bs',
																i));
							}
							log.debug({
								title : 'Success',
								details : '赋值成功：woid = ' + woid
										+ ', warehouse = ' + warehouse
										+ ', clerk = ' + clerk
										+ ', workshop = ' + workshop
										+ ', sublist = ' + rec.getSublist()
							});
						}

					} catch (e1) {
						log.debug({
							title : e1.name,
							details : e1.message
						});
					}
				} else if (type == context.UserEventType.VIEW) {
					var status = context.newRecord.getText({
						fieldId : 'custrecord_yq_ma_status_bansi'
					});
					try {
						if (status == '已批准' || status == 'APPROVED') {
							//绝对路径是文件柜完整路径
							form.clientScriptModulePath = '/SuiteScripts/Bansi_Scripts/lib/BsPrintMAO.js';
							form.addButton({
								id : 'custpage_print_mao',
								label : '打印领料单',
								functionName : 'bs_print_mao'
							});

							log.debug({
								title : 'Success',
								details : '添加-打印领料单-按钮成功！'
							});
						}
					} catch (ex) {
						log.debug({
							title : ex.name,
							details : ex.message
						});

					}
				}
			}

			function beforeSubmit(context) {
				if (context.type != context.UserEventType.CREATE)
					return;

			}

			return ({
				beforeLoad : beforeLoad,
				beforeSubmit : beforeSubmit
			});
		});
```
####  BsPrintMAOSuitelet.js
```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType Suitelet
 * @author YQ12681
 */
////////////////////Description////////////////////
//
//此Suitelet接收请求参数领料单Id(maoId)
//并输出PDF预览响应
//
//////////////////////////////////////////////////
define(
		[ 'N/record', 'N/render' ],
		function(record, render) {
			function onRequest(params) {
				//解析url参数 maoid
				try {
					var maoIdArray = params.request.parameters.maoid.split('_');
					log.debug({
						title : 'Success',
						details : 'maoId参数获取成功 maoIdArray: ' + maoIdArray
					});

				} catch (e) {
					log.debug({
						title : e.name,
						details : e.message
					});

				}
				var xmlStr = '';
				xmlStr += '<?xml version="1.0"?><!DOCTYPE pdf PUBLIC "-//big.faceless.org//report" "report-1.1.dtd">';
				xmlStr += '<pdf>';

				//加载领料申请记录
				for (var i = 0; i < maoIdArray.length; i++) {
					var rec = record.load({
						type : 'customrecord_yq_material_apply_bansi',
						id : maoIdArray[i]
					});
					var maoNumber = rec.getText('name');

					//xml定义打印模板

					xmlStr += '<head>';
					xmlStr += '</head>';
					xmlStr += '<body padding="0.5in 0.5in 0.5in 0.5in" size="Letter">';
					xmlStr += '<table style="width: 100%; font-size: 10pt;">';
					xmlStr += '<tr><td>' + maoNumber + '</td></tr>';
					xmlStr += '</table></body>';

				}
				xmlStr += '</pdf>'
				//xml转成pdf文件
				var pdfFile = render.xmlToPdf({
					xmlString : xmlStr
				});

				//输出文件
				params.response.writeFile({
					file : pdfFile,
					isInline : true
				});

			}

			return {
				onRequest : onRequest
			};
		});
```



### A.2 领料单批量打印源码

#### MaterialApplyOrderPrintSuiteletBS.js

```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType Suitelet
 * @author YQ12681 Bansi ZHU
 */
// //////////////////Function Decription////////////////////
// name: 领料单批量打印
// 
// #1 画表单：包含状态筛选字段（关闭，打开），单据子列表
// #2 创建1个已保存领料单搜索，将结果加载到新页面
// #3 新页面可以根据筛选字段动态加载领料单记录
//
// currentRecord不能用在suitelet中,也不能用以下模块化方式调用加载了
// currentRecord的ClientScript
// @NAmdConfig /SuiteScripts/Bansi_Scripts/configuration.json
//
// ////////////////////////////////////////////////////////
define([ 'N/ui/serverWidget', 'N/search', 'N/record', 'N/render' ],
/**
 * @param {serverWidget} ui
 * @param {search} search
 * @param {record} record
 * @param {render} render
 */
function(ui, search, record, render) {
	function onRequest(context) {
		// if method = GET，调用Suitelet输出或刷新图形界面
		if (context.request.method === 'GET') {
			// #1 新建表单
			var form = ui.createForm({ title : 'Bansi 领料单批量打印' });

			try {
				//两种调用客户端脚本的方式
				//form.clientScriptFileId = 30
				//相对路径 相对当前文件同目录
				form.clientScriptModulePath = './MAOPrintStatusFieldChangedBs.js';
				log.debug({ title : 'Success',
				details : '加载脚本customscript_maoprint_statusfield_bs成功' });
			} catch (ea) {
				log.debug({ title : ea.name,
				details : ea.message });
			}

			var status = context.request.parameters.status;

			// 添加筛选条件字段
			var statusField = form.addField({ id : 'custpage_status_bs',
			label : '状态',
			type : ui.FieldType.SELECT });
			statusField.addSelectOption({ value : 'None_Approved',
			text : '未批准', });

			statusField.addSelectOption({ value : 'Approved',
			text : '已批准',
			isSelected : true });
			// 以URL参数，给字段初始化赋值
			if (status == 'Approved')
				statusField.defaultValue = 'Approved';
			else if (status == 'None_Approved')
				statusField.defaultValue = 'None_Approved';

			//form.clientScriptModulePath = './BsBatchPrintMAO.js';

			// 添加打印按钮
			// Suitelet不能调用含currentRecord模块的脚本,可以用http模块

			/*form.addButton({
				id : 'custpage_print',
				label : '批量打印',
				functionName : 'bs_batch_print_mao'
			});
			*/
			form.addSubmitButton({ label : '批量打印' });

			// 创建子列表
			var slt = form.addSublist({ id : 'custpage_slt',
			label : '领料单',
			type : 'LIST' });

			try { // Adds a Mark All and an Unmark All button to a
				// LIST type of sublist.
				slt.addMarkAllButtons();
				log.debug({ title : 'Success',
				details : '添加批量标记按钮正常' });
			} catch (e1) {
				log.error({ title : e1.name,
				details : e1.message });
			}

			slt.addField({ id : 'custpage_checkbox',
			label : '选择',
			type : ui.FieldType.CHECKBOX });
			slt.addField({ id : 'custpage_mtl_id',
			label : '领料单Id',
			type : ui.FieldType.TEXT });
			slt.addField({ id : 'custpage_mtl_num',
			label : '领料单号',
			type : ui.FieldType.TEXT });
			slt.addField({ id : 'custpage_mtl_date',
			label : '领料申请日期',
			type : ui.FieldType.DATETIMETZ });
			slt.addField({ id : 'custpage_mtl_dept',
			label : '领料部门',
			type : ui.FieldType.TEXT });
			slt.addField({ id : 'custpage_mtl_owner',
			label : '领料人',
			type : ui.FieldType.TEXT });

			// 创建筛选器
			try {
				var flt = [];
				if (status == 'Approved' || !status) {

					/*
					 * flt.push(search.createFilter({ name :
					 * 'customsearch_bansi_mtl_search',//必填 join : null,
					 * operator : search.Operator.ANYOF, formula : [
					 * 'value:3' ] }));
					 */
					flt.push([ 'custrecord_yq_ma_status_bansi', search.Operator.ANYOF, '3' ]);

					log.debug({ title : 'Success',
					details : 'create filter success when status id = Approved ' + flt });
				} else if (status == 'None_Approved') {
					/*
					 * flt.push(search .createFilter({ name :
					 * 'customsearch_bansi_mtl_search', join : null,
					 * operator : search.Operator.ANYOF, formula : [
					 * 'value:1', 'value:2', 'value:4', 'value:5' ] }));
					 */
					flt.push([ 'custrecord_yq_ma_status_bansi', search.Operator.ANYOF, [ '1', '2', '4', '5' ] ]);
					log.debug({ title : 'Success',
					details : 'create filter success when status = None_Approved ' + flt });
				}
			} catch (es) {
				log.debug({ title : es.name,
				details : es.message });
			}

			// 创建saved search
			try {

				/*
				 * var sr = search.create({ type :
				 * 'customrecord_yq_material_apply_bansi', // filters :
				 * flt, columns : [ 'name',
				 * 'custrecord_yq_apply_date_bansi',
				 * 'custrecord_yq_material_apply_dept_bansi',
				 * 'custrecord_yq_material_apply_requestor' ], title :
				 * 'Customer Bansi MAO Print Search01', id :
				 * 'customsearch_bs_mao_search01' });
				 * 
				 * sr.save();
				 * 
				 * var mysearch = search.load({ id :
				 * 'customsearch_bansi_mtl_search' }); var searchResult =
				 * mysearch.run().getRange({ start : 0, end : 100 });
				 */
				// on demand search, needn't title, id
				var searchResult = search.create(
						{
							type : 'customrecord_yq_material_apply_bansi',
							columns : [ 'internalid', 'name', 'custrecord_yq_apply_date_bansi', 'custrecord_yq_material_apply_dept_bansi',
									'custrecord_yq_material_apply_requestor' ],
							filters : flt
						// filters
						// 可以是search.Filter[]对象，也可以是filterExpression
						// object[]
						}).run();

				log.debug({ title : 'Success',
				details : 'Create or load saved search success' });
			} catch (ess) {
				log.debug({ title : ess.name,
				details : ess.message })
			}
			try {
				if (searchResult) {

					// 遍历的两种写法
					/*
					 * for (var i = 0; i < searchResult.length; i++) {
					 * var mtlNum = searchResult[i].getValue({name:
					 * 'name'});
					 * 
					 * var transDate = searchResult[i] .getValue({name:
					 * 'custrecord_yq_apply_date_bansi'}); if (transDate ==
					 * false) transDate = '2018/12/16 23:23:23'; var
					 * dept = searchResult[i] .getValue({name :
					 * 'custrecord_yq_material_apply_dept_bansi'}); var
					 * owner = searchResult[i] .getValue({name :
					 * 'custrecord_yq_material_apply_requestor'});
					 */
					var i = 0;
					searchResult.each(function(result) {
						var mtlId = result.getValue({ name : 'internalid' });
						var mtlNum = result.getValue({ name : 'name' });

						var transDate = result.getValue({ name : 'custrecord_yq_apply_date_bansi' });
						if (transDate == false)
							transDate = '2018/12/16 23:23:23';
						var dept = result.getText({ name : 'custrecord_yq_material_apply_dept_bansi' });
						var owner = result.getText({ name : 'custrecord_yq_material_apply_requestor' });

						// Note that line indexing begins at 0
						// with SuiteScript 2.0
						slt.setSublistValue({ id : 'custpage_mtl_id',
						line : i,
						value : mtlId });
						slt.setSublistValue({ id : 'custpage_mtl_num',
						line : i,
						value : mtlNum });
						slt.setSublistValue({ id : 'custpage_mtl_date',
						line : i,
						value : transDate
						// 如果为空如何处理,现在问题：空导值报错中断
						});
						slt.setSublistValue({ id : 'custpage_mtl_dept',
						line : i,
						value : dept });
						slt.setSublistValue({ id : 'custpage_mtl_owner',
						line : i,
						value : owner });
						i++;
						return true;
					});
					log.debug({ title : 'Success',
					details : 'search result is not null' + i });
				} else
					log.debug({ title : 'Success',
					details : 'search result is null' });

			} catch (esr) {
				log.debug({ title : esr.name,
				details : esr.message });
			}

			// 子列表根据筛选条件动态加载销售订单
			context.response.writePage(form);
		}
		// if method != GET，获取当前页面被选择的领料申请ID，将其详细信息循环打印
		else {
			try {
				var count = context.request.getLineCount({ group : 'custpage_slt' });

				var maoIdArray = [];
				for (var i = 1; i <= count; i++) {
					if (context.request.getSublistValue({ group : 'custpage_slt',
					name : 'custpage_checkbox',
					line : i }) == 'T') {
						var eachMaoId = context.request.getSublistValue({ group : 'custpage_slt',
						name : 'custpage_mtl_id',
						line : i });
						maoIdArray.push(eachMaoId);
					}
				}
				log.debug({ title : 'Success',
				details : 'count = ' + count + ' maoIdArray= ' + JSON.stringify(maoIdArray) });

				var xmlStr = '';
				//xml定义打印模板
				xmlStr += '<?xml version="1.0"?><!DOCTYPE pdf PUBLIC "-//big.faceless.org//report" "report-1.1.dtd">';
				xmlStr += '<pdf>';
				//加载领料申请记录到模板
				for (var i = 0; i < maoIdArray.length; i++) {
					var rec = null;
					try {
						rec = record.load({ type : 'customrecord_yq_material_apply_bansi',
						id : maoIdArray[i] });
					} catch (err) {
						if (err.name == 'RCRD_LOCKED_BY_WF') {
							continue;
						}
					}
					if (rec) {
						var maoNumber = rec.getText('name');
						//模板数据循环
						xmlStr += '<head>';
						xmlStr += '</head>';
						xmlStr += '<body padding="0.5in 0.5in 0.5in 0.5in" size="Letter">';
						xmlStr += '<table style="width: 100%; font-size: 10pt;">';
						xmlStr += '<tr><td>' + maoNumber + '</td></tr>';
						xmlStr += '</table></body>';
					}
				}
				xmlStr += '</pdf>'
				//xml转成pdf文件
				var pdfFile = render.xmlToPdf({ xmlString : xmlStr });

				//输出文件
				context.response.writeFile({ file : pdfFile,
				isInline : true });

			} catch (e1) {
				log.debug({ title : e1.name,
				details : e1.message });
			}

		}

	}
	return { onRequest : onRequest };
});
```

#### MAOPrintStatusFieldChangedBs.js

```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType ClientScript
 * @author YQ12681 Bansi
 */
////////////////////Description//////////////////////
//												
// #1 当领料单批量打印页面字段'状态'变更时，为url添加参数
// 加载1个客制模块：urlUpdate.js
//
////////////////////////////////////////////////////
//
// 下面调用的是本地的urlUpate.js,不需要@NAmdConfig
// define([ 'N/error', 'N/currentRecord', 'N/https', './urlUpdate' ], function(
//
// 下面调用的是/SuiteScripts/Bansi_Scripts/lib下的urlUpate.js,需要@NAmdConfig
// define([ 'N/error', 'N/currentRecord', 'N/https', 'urlUpdate' ], function(
define([ 'N/error', 'N/https', './urlUpdate' ],
/**
 * @param {error} error
 * @param {https} https
 * @param {urlUpdate} urlUpdate
 */
function(error, https, urlUpdate) {

	function fieldChanged(context) {
		try {
			if (context.fieldId == 'custpage_status_bs') {
				var url = window.location.href;
				// 检查url中是否包含&status=参数,如果不存在添加key-value,如果存在替换值
				var words = url.split('?');
				var baseUrl = words[0];
				var oldParamsArray = urlUpdate.urlToParameters(url);
				var newParam = [];
				newParam.push("status");
				var statusValue = context.currentRecord.getValue({ fieldId : 'custpage_status_bs' });
				newParam.push(statusValue);
				var currentParamsArray = urlUpdate.updateParameters(oldParamsArray, newParam);
				var newUrl = urlUpdate.paramsArrayToURL(baseUrl, currentParamsArray);

				window.location.href = newUrl;

				log.debug({ title : 'Success',
				details : 'url重定向成功' + context.fieldId });
			}
		} catch (ex) {
			log.debug({ title : ex.name,
			details : ex.message })
		}
	}

	return { fieldChanged : fieldChanged };
});
```

#### urlUpdate.js

```javascript
/**
 * urlUpdate.js
 * 
 * @NApiVersion 2.x
 * @NModuleScope Public
 * @author YQ12681 Bansi
 */
define(function() {
	function urlToParameters(url) {
		var words = url.split('?');
		var paramArray2 = [];
		if (words.length > 0) {
			var paramStr = words[1];
			var paramArray = paramStr.split('&');

			for (var i = 0; i < paramArray.length; i++) {
				paramArray2.push(paramArray[i].split('='));
			}

		} else {
		}
		return paramArray2;
	}

	function updateParameters(oldParmasArray, newParam) {
		var count = 0;
		for (var i = 0; i < oldParmasArray.length; i++) {
			if (oldParmasArray[i][0] == newParam[0]) {
				oldParmasArray[i][1] = newParam[1];
				count++;
			}
		}
		if (count == 0)
			oldParmasArray.push(newParam);
		return oldParmasArray;
	}

	function paramsArrayToURL(baseUrl, paramsArray) {
		var url = '';
		for (var i = 0; i < paramsArray.length; i++) {
			if (i < paramsArray.length - 1)
				url += paramsArray[i][0] + '=' + paramsArray[i][1] + '&';
			else
				url += paramsArray[i][0] + '=' + paramsArray[i][1];
		}
		return baseUrl + '?' + url;
	}

	return {
		urlToParameters : urlToParameters,
		updateParameters : updateParameters,
		paramsArrayToURL : paramsArrayToURL
	};
});
```

