---
Description: The BreakConnect method releases the pin from a connection.
ms.assetid: a1f299e1-30bf-4d55-84cf-73acccf38151
title: CBasePin.BreakConnect method
ms.technology: desktop
ms.prod: windows
ms.author: windowssdkdev
ms.topic: article
ms.date: 05/31/2018
---

# CBasePin.BreakConnect method

The `BreakConnect` method releases the pin from a connection.

## Syntax


```C++
virtual HRESULT BreakConnect();
```



## Parameters

This method has no parameters.

## Return value

Returns S\_OK.

## Remarks

This method is called during pin disconnection by the [**CBasePin::Disconnect**](cbasepin-disconnect.md) method. It is also called during a connection attempt if the [**CBasePin::CheckConnect**](cbasepin-checkconnect.md) method fails.

This method must free any resources that were obtained by the **CheckConnect** method. For example, if **CheckConnect** allocates memory, `BreakConnect` should free the memory. If **CheckConnect** queries the connecting pin for an interface, `BreakConnect` should free the interface.

Note that `BreakConnect` can be called without a corresponding call to **CompleteConnect**. Therefore, you cannot assume that **CompleteConnect** has been called previously.

## Requirements



|                    |                                                                                                                                                                                            |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Header<br/>  | <dl> <dt>Amfilter.h (include Streams.h)</dt> </dl>                                                                                  |
| Library<br/> | <dl> <dt>Strmbase.lib (retail builds); </dt> <dt>Strmbasd.lib (debug builds)</dt> </dl> |



## See also

<dl> <dt>

[**CBasePin Class**](cbasepin.md)
</dt> </dl>

 

 



