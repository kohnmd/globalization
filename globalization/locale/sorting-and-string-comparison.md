---
title: Sorting and string comparison
description: In the world-ready applications, the alphabetical order can vary among languages, and the conventions for sequencing items can also be quite different
ms.assetid: 9b85cf10-c98b-4e67-b8cf-beb0041782a7
ms.date: 01/24/2017
---
# Sorting and string comparison

Each language has a unique set of rules (sometimes more than one set of rules) for how language strings should be "sorted" or "collated" into an ordered list.
User expectations for those lists may vary based on where the information is presented, such as spreadsheets, tables, dictionaries, and telephone books.
And sometimes, such as for the file system or a web URL, machine consistency is more important than rules for any specific human language.
End users don’t really think about sorting at all until there’s something wrong with it, and then can’t find the specific information they need and assume it’s not there.

In Swedish, for example, some vowels with an accent sort after "Z," whereas in other European countries the same accented vowel comes right after the non-diacritic vowel.
In Hawaiian, all the vowels come first.
Languages that include characters outside the Latin script have special sorting rules.
Asian languages have several different sort orders depending on phonetics, radical order, number of pen strokes, and so on.
Phonetic order can depend on context, such as a phone directory or a dictionary.

One important case where code frequently runs into problems is supporting users who use the Turkish alphabet.
For these languages, there are four variation of the character i and they don’t map to the same casings as English or other languages.
This is called the Turkish-I problem.

| Language | Lower Case Form  | Upper Case Form  |
|-------------------| -----------------| -----------------|
|English            | i (\\u0069)    | I (\\u0049)    |
|Turkic undotted i  | ı (\\u0131)    | I (\\u0049)    |
|Turkic dotted i    | i (\\u0069)    | İ (\\u0130)    |

String sorting and comparison are language-specific.
Even within languages based on the Latin script, there are different composition and sorting rules.
Thus, you should not rely on code points to do proper sorting and string comparison.

## Choosing a sort method – for humans or computers?

When designing your application, you must first decide whether it makes sense to use linguistic or ordinal sorting. Linguistic sorting is used when you need to apply the sorting rules of a particular culture to a set of strings that will be displayed to a human, such as a list of music tracks. Ordinal sorting sorts using binary equivalency and should be used when comparing things that will be processed internally by the software, such as registry keys, environmental variables, etc.

## User experience

Users choose their sort order by setting their language in the operating system. When the chosen language has more than one sort order, the user may specify which of the available sort orders they prefer.

For web applications, there may be built in settings (such as on [SharePoint Online](https://support.office.com/article/Change-regional-settings-for-a-site-e9e189c7-16e3-45d3-a090-770be6e83c1a?ui=en-US&rs=en-US&ad=US&fromAR=1)) or the application may depend on the browser’s language settings via the HTTP\_ACCEPT\_LANGUAGE string.

## Software Challenges and Solutions

Because each language sorts uniquely and sorting rules tend to be complex, implementing sorting correctly is difficult. Fortunately, Windows, .NET, and SQL Server provide sorting functionality that applications can depend on, making life easy for informed developers. Applications only have difficulty with sorting when they try to implement it for themselves, for example to provide some backward compatibility when platform sorting code gets an update, or when the application wants to do a "better job" than what the platform can provide (maybe a platform update is needed, but the next release is years away).

## Sorting and comparing in Win32 Code

> [!NOTE]
> Examples in this topic use LCID to identify locales.
> Microsoft has been migrating toward the use of locale names instead of locale identifiers since Windows Vista.
> Any application that runs only on Windows Vista and later should use the ...Ex versions of most NLS APIs,
> which use a locale name to identify a locale instead of an LCID.
> See [Locale names and LCID deprecation](locale-names.md) for more information.

You can take advantage of the support for sorting that the operating system provides whenever you represent your data in a list-view control that has the sorting attribute set. Edit controls will sort items by following the linguistic sorting rules specific to the currently selected user locale. But if you want to do your own sorting, the real solution depends on your needs.

If you are sorting only a small number of elements, the easiest and quickest solution is to use string comparison techniques.
The [CompareStringEx()](https://msdn.microsoft.com/library/windows/desktop/dd317761(v=vs.85).aspx) Win32 functions compare two character strings in a case-sensitive or case-insensitive manner, respectively, following the sorting rules and standards that are defined for the specified user locale.
The CompareStringEx function compares two strings by looking for linguistic differences per the specified language rules.
This function returns whether String A is greater than, equal to, or less than String B.
For example, the function determines that "abcz" is greater than "abcdefg" and returns a value indicating that "abcz" is greater than "abcdefg."

- Use CompareStringEx and LCMapStringEx to compare or sort information for a given locale
- If you are comparing data against a pre-defined list, consider using CompareStringOrdinal.
- Always use the NORM\_LINGUISTIC\_CASING flag (unless you need CompareStringOrdinal, in which case there is no flag). NORM\_LINGUISTIC\_CASING ensures correct behavior for the Turkish I scenario.

If you are sorting a large number of elements it may be faster to sort long lists of words by retrieving sort keys with LCMapStringEx instead of using CompareStringEx.
Or even better, to use a database.

Applications using sort keys follow this general model:

1. Generate a sort key for each new element.
2. Compare strings by comparing the value of their sort keys using [memcmp](/cpp/c-runtime-library/reference/memcmp-wmemcmp).
3. Store the resulting sort key values along with your string to avoid extra processing time to evaluate the linguistic properties of each string each time they are compared.

A sort key is a composite representation of a sort element (see the figure below).
Each component of the sort key has a different weight, depending on the sort rules associated with the locale.
Elements are sorted based on script (that is, all Greek characters sort together, all Cyrillic characters sort together, and so on); alphanumeric and symbolic characters; diacritics; case; and other special rules, such as two characters sorting as a single character (for example, the Spanish "CH").

### Using sort keys

Once the search engine has a sorted list of words, it can quickly look for matching strings using a binary search algorithm.
Using sort keys is very convenient because they automatically handle language-specific and locale-specific sorting preferences.
One example includes matching ligatures.
(In English, "Æ" is equivalent to "AE," but this is not true in Danish.)
Another example involves distinguishing between single letters and two-character combinations.
(In some Spanish locales, "l" and the combination "ll" are distinct letters.)

Some features of linguistic sorting are:

- A language’s writing system influence the sort order of the language.
  For example, a sort order for Russian would be based on Cyrillic letters and possibly diacritics, but a sort order for Japanese might be based on the number of strokes it takes to draw a character.

- Linguistic sort orders are different than the code point order.

- Languages that use the same script often have different linguistic sort orders.

- A sorting element (such as a character) can be the combination of more than one Unicode code point.
  For example, \\u0B95 + \\u0BCD + \\u0BB7 = Tamil Ksha, a single sorting element in Tamil (<span lang="ta">க</span> + ் + <span lang="ta">ஷ</span> = <span lang="ta">க்ஷ</span>).

The best thing to remember is that by using Windows system support, all of these sorting issues are automatically handled for you.

### Sort version

Regardless of whether your index used CompareStringEx directly or generated a list of sort keys with LCMapStringEx, the results of that sort order are tied to the current set of rules used by the operating system.
If the sort order changes in the operating system, then your index may need to be recalculated.
Typically this is not a concern for short-lived processes, however if your application persists your index over operating system updates, such as in a file on disk, then you may need to consider this scenario.

For example, new characters could be added to Unicode and end up sorting between character A and character B.
Previously that character would have been unknown, but now it needs to sort in the correct location.
If your data contained that character and you tried to look it up, the search algorithm could get confused by finding a record with this new character in the wrong location.
For a binary search or tree, this could cause entire sections of the data to become inaccessible if the search then started looking in the wrong direction for the requested data.

Another example would be sorting in a language for which Windows does not have full support.
A future update to Windows with improved language support could provide correct ordering for that language, invalidating any previously generated indices for that language.

To mitigate against this, applications can call [GetNLSVersionEx](/windows/win32/api/winnls/nf-winnls-getnlsversionex) with the COMPARE\_STRING flag and store the results along with the index.
The important bits of the NLSVERSIONINFOEX structure to compare are the dwNLSVersion, which will be updated if the general sorting rules change, and guidCustomVersion, which is effectively a GUID for each language and will change if the rules for a specific language change.

## Sorting and comparing in .NET Code

The Array class provides an overloaded method that allows you to sort arrays based on the CultureInfo.CurrentCulture property.
In the following example, an array of three strings is created.
First, the [CurrentCulture](/dotnet/api/system.threading.thread.currentculture) is set to "en-US" and the method is called.
The resulting sort order is based on sorting conventions for the "en-US" culture.
Next, the [CurrentCulture](/dotnet/api/system.threading.thread.currentculture) is set to "da-DK" and the method is called again.
Notice how the resulting sort order differs from the "en-US" results because the sorting conventions for the "da-DK" culture are used.

```csharp
using System;
using System.Threading ;
using System.Globalization ;

public class SortKeySample 
{
  public static void Main(String[] args )
  {
    String str1 = "Apple";
    String str2 = "Æble";

    // Set the CurrentCulture to "da-DK".
    CultureInfo dk = new CultureInfo ("da-DK");
    Thread.CurrentThread.CurrentCulture = dk ;

    // Create a culturally sensitive sort key for str1.
    SortKey sc1 = dk.CompareInfo.GetSortKey ( str1);

    // Create a culturally sensitive sort key for str2.
    SortKey sc2 = dk.CompareInfo.GetSortKey ( str2);

    // Compare the two sort keys and display the results.

    int result1 = SortKey.Compare(sc1, sc2);
    Console.WriteLine ("\nWhen the CurrentCulture is \"da-DK\",\n");

    Console.WriteLine ("\the result of comparing {0} with {1} is: {2}", str1, str2, result1);

    // Set the CurrentCulture to "en-US".
    CultureInfo enus = new CultureInfo ("en-US");
    Thread.CurrentThread.CurrentCulture = enus ;

    // Create a culturally sensitive sort key for str1.
    SortKey sc3 = enus.CompareInfo.GetSortKey (str1);
    
    // Create a culturally sensitive sort key for str1.
    SortKey sc4 = enus.CompareInfo.GetSortKey (str2);
    
    // Compare the two sort keys and display the results.
    int result2 = SortKey.Compare(sc3, sc4);
    
    Console.WriteLine (
    "\nWhen the CurrentCulture is \"en-US\",\n");
    Console.WriteLine ("\the result of comparing {0} with {1} is: {2}",str1, str2, result2);
  }
}
```

This code produces the following output:

```console
    When the CurrentCulture is "da-DK",
    the result of comparing Apple with Æble is: -1

    When the CurrentCulture is "en-US",
    the result of comparing Apple with Æble is: 1
```

### Using sort keys in .Net

Once again, sort keys are used to support sorting methods that vary from one culture to another.
According to sort-key methodology, each character in a string is given several categories of sort weights, including alphabetic, case, diacritic and other language-specific weights.
A sort key serves as the repository of these weights for a particular string.
For example, a sort key might contain a string of alphabetic weights, followed by a string of case weights, and so on.

In the .NET Framework, the [SortKey](/dotnet/api/system.globalization.sortkey) class maps strings to their sort keys and vice versa.
You can use the [CompareInfo.GetSortKey](/dotnet/api/system.globalization.compareinfo.getsortkey) method to create a sort key for a string that you specify.
The resulting sort key for a specified string is a sequence of bytes that can differ depending upon the CurrentCulture and the CompareOptions that you specify.
For example, if you specify IgnoreCase when creating a sort key, a string comparison operation using the sort key will ignore case.

After you create a sort key for a string, you can pass it as a parameter to methods provided by the [SortKey](/dotnet/api/system.globalization.sortkey) class.
The [SortKey.Compare](/dotnet/api/system.globalization.sortkey.compare) method allows you to compare sort keys.
Because [SortKey.Compare](/dotnet/api/system.globalization.sortkey.compare) performs a simple byte-by-byte comparison, it is much faster than using [String.Compare](/dotnet/api/system.string.compare).
In applications that are sorting-intensive, you can improve performance by generating and storing sort keys for all the strings that the application uses.
When a sort or comparison operation is required, you can use the sort keys rather than the strings.

The previous code example creates sort keys for two strings when the CurrentCulture is set to "da-DK".
It compares the two strings using the SortKey.Compare method and displays the results.
 [SortKey.Compare](/dotnet/api/system.globalization.sortkey.compare) method returns a negative integer if string1 is less than string2, zero (0) if string1 and string2 are equal, and a positive integer if string1 is greater than string2.
Next, the [CurrentCulture](/dotnet/api/system.threading.thread.currentculture) is set to "en-US" culture, and sort keys are created for the same strings.
The sort keys for the strings are compared and the results are displayed.
Notice that the sort results differ based upon the [CurrentCulture](/dotnet/api/system.threading.thread.currentculture).
Although the results of the following code example are identical to the results of comparing these strings in the [String.Compare](/dotnet/api/system.string.compare) example given earlier, the [SortKey.Compare](/dotnet/api/system.globalization.sortkey.compare) method is faster than the [String.Compare](/dotnet/api/system.string.compare) method.

### String normalization

Some Unicode characters have multiple equivalent binary representations consisting of sets of combining and/or composite Unicode characters. The Unicode standard defines a process called normalization that returns one binary representation when given any of the equivalent binary representations of a character.

For example, the glyph ä may be canonically represented by the following sets of code points:

|Characters  | Code Points        |
|------------| -------------------|
|ä           | \\u00E4            |
|a + ̈        |  \\u0061 + \\u0308 |

- Use the [IsNormalized](/dotnet/api/system.string.isnormalized) method to test whether a string is normalized to a particular normalization form.
- Use the [Normalize](/dotnet/api/system.string.normalize) method to create a string that is normalized to a particular normalization form.
- Many computer systems and standards tend to the Unicode Normalization Form C representation, however others lean toward Form D. If your applications gets input from more than one system it may see data in either form.

## Security Considerations

Typically it is not desirable for security decisions based on string comparisons to change if the machine’s CurrentCulture is changed. Changes in sorting behavior between languages could lead to security holes. In those cases ordinal comparisons are recommended instead of using a linguistic sensitive comparison.

Often the ordinal comparisons required for security comparisons should be exact matches, however some uses traditionally have allowed case-insensitive behavior. Though not linguistically accurate or complete, the ordinal comparisons allow selecting case-insensitive behavior when required.

## What if I need culture-insensitive sorting?

There are several situations where you would want your sort order not to vary from culture to culture (for example, for maintaining ordered lists that require fast searching).
The article [Performing Culture - String Operations in Collections](/dotnet/standard/globalization-localization/performing-culture-insensitive-string-operations-in-collections) has lots of information on this subject.

## How do I implement collations in SQL Server?

See the [Collation and Unicode Support](/sql/relational-databases/collations/collation-and-unicode-support) article.

## How do I verify that sorting works correctly in my code?

If you are relying on Windows, .Net, SQL or another of established framework for string sorting/comparisons, there is typically little benefit in trying to comprehensively test sorting rules for each locale but you'll still want to validate that the sorting order correctly follows the user’s regional and language settings using sample data similar to below:

| Expected order English-US | Expected order French - France | Expected order Spanish - Spain  |
| ----- | ----- | ----- |
| cote    | cote       | cote |
| coté    | __*côte*__ | coté  |
| côte    | __*coté*__ | côte  |
| côté    | côté       | côté  |
| Namibia | Namibia    | Namibia  |
| ñandú   | ñandú      | __*número*__  |
| ñú      | ñú         | __*ñandú*__  |
| número  | número     |__*ñú*__  |

## Market-specific sorting examples

### East Asian sorting

It’s common for East Asian languages to having more than one sorting order.
You’ll notice when you customize the user locale for some East Asian languages, there’s additional options that you might not have seen before for other languages.
East Asian languages, such as Chinese and Japanese, can be sorted by either pronunciation or stroke count, and both are widely used for different purposes.

For those unfamiliar with East Asian sorting, a brief explanation may be needed.
Technically, what is commonly referred to "Stroke Count" or "Stroke" sort order is a radical-and-stroke sorting used for non-alphabetic writing systems such as Chinese and the logographic systems that derived from Chinese, whose thousands of symbols defy alphabetic-style ordering.

In these languages, common components of characters are identified called "radicals."
Characters are then grouped by their primary radical, then ordered by the number of pen strokes within each radical.
When there is no obvious radical or more than one radical, convention governs which is used for sorting.
For example, the character for "mother" (<span lang="ja">媽</span>) is sorted as a thirteen-stroke character under the three-stroke primary radical (<span lang="ja">女</span>).

The radical-and-stroke system is cumbersome compared to an alphabetical system in which there are a few characters, all unambiguous.
The choice of which components of a logograph comprise separate radicals and which radical is primary is not always clear-cut.
As a result, the logographic languages often supplement radical-and-stroke ordering with alphabetic sorting of a phonetic conversion of the logographs.
For example, the kanji word Tōkyō (<span lang="ja">東京</span>), the Japanese name of Tokyo can be sorted as if it were spelled out in the Japanese characters of the hiragana syllabary as "to-u-ki-yo-u" (<span lang="ja">とうきょう</span>), using the conventional sorting order for these characters, this is "phonetic" sorting.

### Japanese sorting

For sorting Japanese names, some features have phonetic name support, such as the Japanese Outlook Address Book.
The following names are sorted correctly, even though the phonetic name info weren’t available from the user.

- <span lang="ja">麻生太郎
- おかわりくん
- 川端康成
- どらえもん
- どらえもん２</span>

The second list is acceptable when phonetic sorting is not available and has to rely on user input their phonetic names.

- <span lang="ja">おかわりくん
- どらえもん２
- 麻生太郎
- 川端康成
- どらえもん</span>

### Simplified Chinese sorting

Simplified Chinese generally uses the Pinyin pronunciation sort order, but other sort orders are commonly offered to end-users.
Windows also provides the Surname sorting option for people’s names.

### Traditional Chinese sorting

Taiwan has one system based on the pronunciation in Bopomofo order and this is the prevalent system for collating Chinese in Taiwan.
Whereas in Hong Kong SAR, users commonly sort by using stroke order.

### Korean sorting

Korean is sorted based on the Hangeul pronunciation of the Hangeul and Hanja code-points.

## Find and replace

Find and replace can be important feature that uses sorting and compare principles.
Different applications have different levels of support for find and replace.
In addition to the core support, applications may choose to add support for additional language find and replace features such as:

- Match Diacritics (languages with accents or other diacritics)

- Match Kashida (languages written the Arabic script)

- Match Alef Hamza (languages written the Arabic script)

- Hanja with Phonetic Hangeul (Korean)

- Match Half- and Full-Width Forms (Japanese)

- Sounds Like (Japanese)

**Three things should be noted:**

- These additional features should not break any of the core Find and Replace features.

- Not all these features have an effect on all languages. For example, "Match Case" should not affect Arabic text, as it doesn’t have case.

- Wild Cards should work with all languages

### Match Diacritics

Diacritics do have an effect on some languages on both spelling and meaning of words.
For Arabic and Hebrew diacritics are not usually used on all words especially in magazines and newspapers and many books.
However, they are usually used to remove ambiguity in places where it might be introduced and the reader cannot tell from the context.
Special books fully make use of diacritics and does not rely on the reader knowledge of the language to correctly pronounce the words.

Match Diacritics in the Find and Replace feature is used to ignore or consider diacritics when searching for text.
If, for example, Match Diacritics was checked, diacritics will not be ignored and found words must match the exact type and location of all diacritics in the searched text.
On the other hand, if Match Diacritics was unchecked, diacritics will be ignored and not considered when matching found words with searched text in the Find and Replace dialog.
Again, using diacritics in the Find and Replace dialog will not be ignored even if the Match Diacritics is unchecked.

|  Search For |  Setting                     |  Search Finds |
|-------------| -----------------------------| --------------|
|  é          |  Match Diacritics selected   |  é            |
|  é          |  Match Diacritics unselected |  e, é         |
|  <span lang="he">שׁ</span>          |  Match Diacritics selected   |   <span lang="he">שׁ</span>           |
|  <span lang="he">ש</span>          |  Match Diacritics unselected |  <span lang="he">ש, שׁ, שׂ</span>      |

### Match Kashida

Kashida is mainly used in Arabic text for the sole purpose of text justification in both left and right sides.
It, however, may also be used by some for enhancing visual appearance of some words.
Kashida should have no effect on spelling or meaning of any word it is inserted to.
You may choose to add one or more kashidas between any connected characters.

Match Kashida in Find and Replace functionality is used to ignore or consider kashida when searching for text.
If, for example, Match Kashida was checked, kashidas will not be ignored in your search or replacement.
Hence Find and Replace will only find words whose kashidas location and number exactly match those of the text typed in Find and Replace.
On the other hand, if Match Kashida was unchecked, kashidas will simply be ignored and Arabic words such as (<span lang="ar">حليم</span> and<span lang="ar"> حليــــــــــم</span>) would be equivalent and searching the first word would catch both words. However, searching the second one (the one with a kashida) will only find the same word and not the other one (without the kashida).
This is OK since the user would type the kashida in Find and Replace only if he/she really intends to include it in the search.

### Match Alef Hamza

It happens very often that the Arabic letter Alef with Hamza (<span lang="ar">أ</span>) is written as simply Alef without Hamza (<span lang="ar">ا</span>) on top of it with affecting the meaning.
Hence, some applications include "Match Alef Hamza" to selectively choose to ignore or consider the Alef Hamza when searching Arabic text.
For example, if Match Alef Hamza is unchecked, then searching for <span lang="ar">(احمد)</span> or <span lang="ar">(أحمد)</span> will find both <span lang="ar">(احمد)</span> and <span lang="ar">(أحمد)</span>.
On the other hand, if Match Alef Hamza is checked, then the Alef and Alef Hamza must match that of the text used in the Find and Replace dialog.
Therefore, searching for <span lang="ar">(احمد)</span> will not find <span lang="ar">(أحمد)</span>.

### Hanja with phonetic Hangeul

The "Hanja with phonetic Hangeul" option is used to find words written in either the hanja (Han or "Chinese script") or Hangeul (native Korean script).
For example, the terms <span lang="ko">한자</span> and <span lang="ko">漢字</span> both mean "Hangeul."

### Full and half-width forms

This option allows users to search for full- and half-width forms.
This means the pair A (\\u0041) and full-width Ａ (\\uFF21) and the pair <span lang="ja">ァ</span> (\\u30A1) and half-width <span lang="ja">ｦ</span> (\\uFF66) will be considered equivalent characters.
To correctly search for these characters, it is recommended to [normalize](#string-normalization) the strings first.

### Sounds like

This feature allows users to treat characters as equal.
For example, it will treat the characters in the hiragana and katakana scripts as the same.
For example, the following string could be written in the following manner.

English: The spaghetti we had for dinner was delicious.

Japanese: <span lang="ja">夕食に食べたスパゲッティは、美味しかったです。</span>

Hiragana: <span lang="ja">ゆうしょくにたべたすぱげってぃは、おいしかったです。</span>

Katakana: <span lang="ja">ユウショクニタベタスパゲッティハ、オイシカッタデス。</span>

### Wildcard search

Even when user use wildcards, the search should not include partial characters.
Wild card symbols can be even a constituent character or simple character.
This can mean a single character in English or other language if they precede or follow the matched data.
However, for other languages the cases may be more complicated as shown for Thai below.

- Any character?
- Previous one or more @, etc.

|  Search String|  Text  |  Result   |
|---------------| -------| ----------|
| <span lang="th">กี่?</span> | <span lang="th">หกี่ก</span> | Match     |
| <span lang="th">กี?</span> | <span lang="th">หกี่ก</span> | No match  |
| <span lang="th">กี่?</span> | <span lang="th">หกีa</span> | Match     |
| <span lang="th">กี่@</span> | <span lang="th">หกี่ก</span> | Match     |
| <span lang="th">กี@</span> | <span lang="th">หกี่ก</span> | No match  |
| <span lang="th">กี่@</span> | <span lang="th">กี่ก</span>  | Match     |
