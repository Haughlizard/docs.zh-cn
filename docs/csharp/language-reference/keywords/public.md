---
title: public 关键字 - C# 参考
ms.date: 07/20/2015
f1_keywords:
- public
- public_CSharpKeyword
helpviewer_keywords:
- public keyword [C#]
ms.assetid: 0ae45d16-a551-4b74-9845-57208de1328e
ms.openlocfilehash: 19906d7fd0f7d41ef9e4cdaf951c77825e0bbead
ms.sourcegitcommit: 7588136e355e10cbc2582f389c90c127363c02a5
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/14/2020
ms.locfileid: "75713165"
---
# <a name="public-c-reference"></a>public（C# 参考）

`public` 关键字是类型和类型成员的访问修饰符。 公共访问是允许的最高访问级别。 对访问公共成员没有限制，如以下示例所示：

```csharp
class SampleClass
{
    public int x; // No access restrictions.
}
```

有关详细信息，请参阅[访问修饰符](../../programming-guide/classes-and-structs/access-modifiers.md)和[可访问性级别](accessibility-levels.md)。

## <a name="example"></a>示例

在下面的示例中，声明了两个类：`PointTest` 和 `MainClass`。 直接从 `MainClass` 访问 `PointTest` 的公共成员 `x` 和 `y`。

[!code-csharp[csrefKeywordsModifiers#13](~/samples/snippets/csharp/VS_Snippets_VBCSharp/csrefKeywordsModifiers/CS/csrefKeywordsModifiers.cs#13)]

如果将 `public` 访问级别更改为 [private](private.md) 或 [protected](protected.md)，则会收到错误消息：

“PointTest.y”不可访问，因为它受保护级别限制。

## <a name="c-language-specification"></a>C# 语言规范  

有关详细信息，请参阅 [C# 语言规范](/dotnet/csharp/language-reference/language-specification/introduction)中的[声明的可访问性](~/_csharplang/spec/basic-concepts.md#declared-accessibility)。 该语言规范是 C# 语法和用法的权威资料。

## <a name="see-also"></a>另请参阅

- [C# 参考](../index.md)
- [C# 编程指南](../../programming-guide/index.md)
- [访问修饰符](../../programming-guide/classes-and-structs/access-modifiers.md)
- [C# 关键字](index.md)
- [访问修饰符](access-modifiers.md)
- [可访问性级别](accessibility-levels.md)
- [修饰符](index.md)
- [private](private.md)
- [受保护](protected.md)
- [内部](internal.md)
