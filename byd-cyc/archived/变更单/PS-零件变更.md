#### gantang.chgmgmt.eco.ecoitems.PartChangeActionView

```js
var changeStatus = record.data['change.changeStatus'];
if (changeStatus != 'DRAFT' && changeStatus != 'RETURNED') {
	value = "查看"
}
```

```js
var changeStatus = record.data['change.changeStatus'];
if (changeStatus == 'APPROVAL') {
	gridPanel = Ext.create('gantang.chgmgmt.eco.ecoitems.PartChangeInfoViewForUp');
}else if (changeStatus != 'DRAFT' && changeStatus != 	     		'RETURNED' && changeStatus != 'APPROVAL') {
    gridPanel = Ext.create('gantang.chgmgmt.eco.ecoitems.PartChangeInfoViewForUp',{showButtons : false,formReadOnly : true});
}else{
	gridPanel = Ext.create('gantang.chgmgmt.eco.ecoitems.PartChangeInfoView');
}
```

