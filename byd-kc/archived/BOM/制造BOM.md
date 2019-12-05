#### 描述

- 制造BOM承接订单工程BOM的数据

- 可初始化制造BOM的订单车型应该符合如下要求
  1. 订单车型已初始化BOM
  2. 订单车型在指定工厂下未初始化过
  3. 订单车型达到了MBOM状态
  
- 初始化制造BOM时,查询订单车型的BOM数据

  ```
  {partId=7161, level=99, bomType2=PART, isShowDelete=true, bomType=OEBOM, isEffective=true}
  然后删除操作类型是删除的数据
  ```

- preReceivedNo 会记录接受数据的版本号

#### 制造BOM初始化

#####  制造端会接受数据的要求

> fromPa 待接受的BOM行；existPa是根据fromPa 的行号在接受工程下找到的BOM行,existPa不存在的数据可以接受

- existPa.getPreReceivedNo() < fromPa.getVersionNo()
-  existPa.getVersionNo() == 1L && existPa.getActiveStatus().equals(CodeListConstant.ACTIVE_STATUS_DRAFT)

> 删除符合要求的BOM行；如下代码不会生效，应为在查询fromPa时就在内存中删除了操作类型是删除的数据

```java
if (existPa != null) {
			//当MBOM中只有一版数据 ,并且数据停留在草稿阶段
			if(existPa.getPreVersionId() == null && existPa.getActiveStatus().equals(CodeListConstant.ACTIVE_STATUS_DRAFT)) {
				//从工程传下来删除的请求，直接删除
				if (CodeListConstant.OPERATION_TYPE_DELETE.equals(fromPa.getOperationType())) {
					this.partAssemblyDao.remove(existPa);
					return true;
				}
			}
		}
```

##### 制造BOM初始化时会复制(初始化)的属性

> 要求
>
> 1. 将工程BOM上的建议来源(采购类别)复制到实际来源上 suggestSourcing  =>  bomAttachRecords.custCraftsAttach.sourcing
> 2. 

```java
		// 映射工厂build
		mapperFactory = new DefaultMapperFactory.Builder().build();
		// partAssembly to partAssembly copy mapper init
		mapperFactory.classMap(PartAssembly.class, PartAssembly.class).exclude("masterPart").exclude("subPart").exclude("employee").exclude("plant")
				.exclude("preVersion").exclude("change").exclude("breakPoint").exclude("id").exclude("effectiveFrom").exclude("effectiveTo")
				.exclude("actualEffectiveFrom").exclude("actualEffectiveTo").exclude("optCounter").exclude("changeId").exclude("breakPointId")
				.exclude("sourceChangeId").exclude("sourceChange").byDefault().register();
		boundMapper = mapperFactory.getMapperFacade(PartAssembly.class, PartAssembly.class);

PartAssembly toPartAssembly = boundMapper.map(srcPartAssembly);
toPartAssembly.setIsLatest(true);
toPartAssembly.setActiveStatus(CodeListConstant.ACTIVE_STATUS_DRAFT);
toPartAssembly.setVersionNo(srcPartAssembly.getVersionNo() + 1);
toPartAssembly.getPartAssemblyExt().setId(null);
toPartAssembly.setVersionNo(1L);
toPartAssembly.setPreReceivedNo(srcPartAssembly.getVersionNo());
toPartAssembly.setBomStatus(null);
toPartAssembly.setPreVersionId(null);
toPartAssembly.setBomType(toBomType());
toPartAssembly.setPlantId(bomInitHolder.get().getPlantId());
toPartAssembly.setFromBomType(bomInitHolder.get().getFromBomType());
toPartAssembly.setBatchNo(bomInitHolder.get().getBatchNo());
if (srcPartAssembly.getBomType().equals(CodeListConstant.BOM_POLICY_MBOM)) {
    toPartAssembly.setSourcing(srcPartAssembly.getSourcing());
} else {
    toPartAssembly.setSourcing(srcPartAssembly.getSuggestSourcing());
}
toPartAssembly.setUsageDesc(srcPartAssembly.getUsageDesc());
toPartAssembly.setUsageValue(srcPartAssembly.getUsageValue());
toPartAssembly.setBomStatus(CustCodeListConstant.BOM_STATUS_PRODUCTION_PREPARE);
toPartAssembly.setFromBomType(srcPartAssembly.getBomType());
```

- 如果BOM行已在制造BOM中存在,并且制造BOM行符合要求,则从已存在BOM行的上复制一下属性到新的BOM行上

```java
newBom.setId(oldBom.getId());
newBom.setOptCounter(oldBom.getOptCounter());
newBom.setVersionNo(oldBom.getVersionNo());
newBom.setPartAssemblyExtId(oldBom.getPartAssemblyExtId());
newBom.getPartAssemblyExt().setId(oldBom.getPartAssemblyExtId());
newBom.setChangeId(oldBom.getChangeId());

newBom.setFilterType(oldBom.getFilterType());
newBom.setLineCode(oldBom.getLineCode());
newBom.setLineName(oldBom.getLineName());

newBom.setStationId(oldBom.getStationId());
if (oldBom.getSourcing() != null) {
    newBom.setSourcing(oldBom.getSourcing());
}
// 如果发生数量变化则工位拆分的数据不保存，下游需要重新拆分
if (newBom.getQuantity().doubleValue() == oldBom.getQuantity().doubleValue()) {
    newBom.setStation1(oldBom.getStation1());
    newBom.setCount1(oldBom.getCount1());
    newBom.setStation2(oldBom.getStation2());
    newBom.setCount2(oldBom.getCount2());
    newBom.setStation3(oldBom.getStation3());
    newBom.setCount3(oldBom.getCount3());
}
```

- 初始化完成后操作
	
  - 修改BOM状态
  
  > 没有作用，生成订单BOM时会换lineNum号，也就是无法更改到工程BOM的BOM行状态
  
  ```
	  bomType.add(CodeListConstant.BOM_POLICY_EBOM);
	bomType.add(CustCodeListConstant.BOM_POLICY_OEBOM);
	  bomType.add(CodeListConstant.BOM_POLICY_MBOM);
	  通过行号(被制造接受的BOM行行号)查找上述3中类型的BOM行，将bomStatus改为生产准备状态(PRODUCTION_PREPARE),如果已经是投产方行状态则不做修改(PRODUCTION_PERMISSION)
	
	```
	
	- 维护mbom同步零件工艺路线库
	
	>   维护mbom同步零件工艺路线库
	> 	 优先级4:全路径 + 上级零件+ 结构区域 + 功能位置
	> 	 优先级3:上级零件
	> 	 优先级2:结构区域 + 功能位置
	> 	 优先级1:通用
	
	```
	// 获得所有优先级,当前bom行涉及 的优先级
	List<Long> routingPriorityList = getPriorityListByPartAssembly(partAssembly);
	// 获得新增所对应的lineType集合， 可能是工序， 或者工艺路线
	List<String> lineTypeList = getLineType(partAssembly);
	
	lineTypeList.stream().forEach(lineType -> {
	// partAssembly to partRoutingList
		List<PartRouting> partAssemblyToPartRouting = partAssemblyToPartRouting(partAssembly, routingPriorityList, lineType);
		for (PartRouting partRouting : partAssemblyToPartRouting) {
	// 将零件工位编码和名称存在lineCode和lineName中，若为空，则不新增
	        if (StringUtils.isNotEmpty(partRouting.getLineName())
	        && StringUtils.isNotEmpty(partRouting.getLineCode())) {
	            partRoutingService.doCreate(partRouting);
	        }
	    }
		partRoutingDao.flush();
});
	```
	
- 维护多领域信息
- 接收变量车型ReceivedVehicle



#### 制造接受

- 制造的待接收文件通过什么方式产生
  - 设计变更单页面->选择制造基地点击发放
  - 分组数据发布单->选择制造基地点击发放
- ？整版数据发布单后不允许在走分组发布单
- ？整版发布单发布时，若还有分组发布单没有发布怎么办，有没有这样的校验

##### 制造接收逻辑

```java
 partAssemblyDao.createQuery().idIn(recIds);
 // 	TODO YANZW 制造BOM可以直接接收平台车型的变更吗？
 if (CustCodeListConstant.CHANGE_VEHICLE_TYPE_ENGINEER.equals(change.getChangeExt().getVehicleType())) {
 // 当接收的变更单为平台车变更单时, 只接收零件结构的数据
 partAssemblyDao.bomTypeEquals(CustCodeListConstant.BOM_POLICY_PART);
 }
 List<PartAssembly> partAssemblies = partAssemblyDao.list(); // 要接收的BOM数据
```

###### 订单工程BOM作废数据的处理

- 制造端如果没有接收这条数据则不用处理
- 制造端如果是草稿版本则直接删除,并回退到上一版本
- 制造端如果是生效版本则生成一条草稿版的数据,等待发布单发布

// 没有前一版本，把 sourceId 修改设置成 id
		if (obj.getPreVersionId() == null && obj.getId() != null) {
			obj.setSourceId(obj.getId());
		}

#### 制造数据维护

##### 工序和工作中心

- 工序从制造基础数据->车间布局中选择
- 一个工作中心对应多个工序，一个工序最多对应一个工作中心。选择工序带出工作中心，若工序没有工作中心,则不要用空白覆盖工作中心
- 选择工作中心不会反选出工序



