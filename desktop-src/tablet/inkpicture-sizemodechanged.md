---
Description: 'Occurs after the SizeMode property of the InkPicture control has been changed.'
ms.assetid: 'ae56b5a2-e3e2-468c-a572-a9b46eb1d39d'
title: 'InkPicture.SizeModeChanged event'
---

# InkPicture.SizeModeChanged event

Occurs after the [**SizeMode**](inkpicture-sizemode.md) property of the [InkPicture](inkpicture-control-reference.md) control has been changed.

## Syntax


```C++
void SizeModeChanged(
  [in]�InkPictureSizeMode NewMode,
  [in]�InkPictureSizeMode OldMode
);
```



## Parameters

<dl> <dt>

*NewMode* \[in\]
</dt> <dd>

The new state of the [InkPicture](inkpicture-control-reference.md) control, based on the new value of the [**SizeMode**](inkpicture-sizemode.md) property.

</dd> <dt>

*OldMode* \[in\]
</dt> <dd>

The old state of the [InkPicture](inkpicture-control-reference.md) control, based on the old value of the [**SizeMode**](inkpicture-sizemode.md) property.

</dd> </dl>

## Return value

This event does not return a value.

## Remarks

This event method is defined in the **\_IInkPictureEvents** interface. The **\_IInkPictureEvents** interface implements the [**IDispatch**](ebbff4bc-36b2-4861-9efa-ffa45e013eb5) interface with an identifier of DISPID\_IPESizeModeChanged.

## Requirements



|                                     |                                                                                                                     |
|-------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Minimum supported client<br/> | Windows�XP Tablet PC Edition \[desktop apps only\]<br/>                                                       |
| Minimum supported server<br/> | None supported<br/>                                                                                           |
| Header<br/>                   | <dl> <dt>Msinkaut.h (also requires Msinkaut\_i.c)</dt> </dl> |
| Library<br/>                  | <dl> <dt>InkObj.dll</dt> </dl>                               |



## See also

<dl> <dt>

[InkPicture](inkpicture-control-reference.md)
</dt> </dl>

�

�



