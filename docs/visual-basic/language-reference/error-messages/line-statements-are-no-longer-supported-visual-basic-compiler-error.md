---
title: 不再支持“Line”语句（Visual Basic 编译器错误）
ms.date: 07/20/2015
f1_keywords:
- bc30830
- vbc30830
helpviewer_keywords:
- BC30830
ms.assetid: 4734bc1d-882e-4555-b498-1f1ec0399d16
ms.openlocfilehash: a3d243f39f3fc45ca6b1ba0d26892d4c3db56f59
ms.sourcegitcommit: f8c270376ed905f6a8896ce0fe25b4f4b38ff498
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 06/04/2020
ms.locfileid: "84397293"
---
# <a name="line-statements-are-no-longer-supported-visual-basic-compiler-error"></a>不再支持“Line”语句（Visual Basic 编译器错误）
不再支持行语句。 文件 i/o 功能提供为 `Microsoft.VisualBasic.FileSystem.LineInput` ，图形功能作为提供 `System.Drawing.Graphics.DrawLine` 。  
  
 **错误 ID：** BC30830  
  
## <a name="to-correct-this-error"></a>更正此错误  
  
1. 如果执行文件访问，请使用 `Microsoft.VisualBasic.FileSystem.LineInput` 。  
  
2. 如果执行图形，则请使用 `System.Drawing.Graphics.Drawline`。  
  
## <a name="see-also"></a>另请参阅

- <xref:System.IO>
- <xref:System.Drawing>
- [使用 Visual Basic 访问文件](../../developing-apps/programming/drives-directories-files/file-access.md)
