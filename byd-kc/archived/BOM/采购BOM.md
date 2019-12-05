#### 采购BOM-同步

- 数据来源于订单车型工程BOM(订单车型要走过整车数据发布单,订单车型走过整车数据发布单后状态会变成MBOM)

  ```java
  params.put("bomType", CustCodeListConstant.BOM_POLICY_OEBOM);
  params.put("bomType2", CustCodeListConstant.BOM_POLICY_PART);
  params.put("isEffective", "on");
  params.put("level", "99");
  params.put("notShowDelete", false);
  params.put("partId", partId);
  params.put("otherInfo", new String[]{});
  custOrderEBomService.doSharedBomQuery(new BomHeader(params));
  ```

- 只要采购件和客供件
- 只要供应商生效的零件

## 与业务无关

- 采购BOM按生效时间排逆序

