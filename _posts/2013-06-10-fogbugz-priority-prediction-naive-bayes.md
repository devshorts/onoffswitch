---
layout: post
title: Automatic fogbugz triage with naive bayes
date: 2013-06-10 08:00:39.000000000 -07:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
tags:
- c#
- classification
- fogbugz
- machine learning
- naive bayes
meta:
  _edit_last: '1'
  _su_rich_snippet_type: none
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1558918892;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4027;}i:1;a:1:{s:2:"id";i:4170;}i:2;a:1:{s:2:"id";i:3847;}}}}

permalink: "/2013/06/10/fogbugz-priority-prediction-naive-bayes/"
---
<p>At my work we use fogbugz for our bugtracker and over the history of our company's lifetime we have tens of thousands of cases.  I was thinking recently that this is an interesting repository of historical data and I wanted to see what I could do with it.  What if I was able to predict, to some degree of acuracy, who the case would be assigned to based soley on the case title?  What about area? Or priority? Being able to predict who a case gets assigned to could alleviate a big time burden on the bug triager.</p>
<p>Thankfully, I'm reading "Machine Learning In Action" and came across the naive bayes classifier, which seemed a good fit for me to use to try and categorize cases based on their titles.  Naive bayes is most famously used as part of spam filtering algorithms.  The general idea is you train the classifier with some known documents to seed the algorithm.  Once you have a trained data set you can run new documents through it to see what they classify as (spam or not spam).</p>
<p>For those who've never used Fogbugz, let me illustrate the data that's available to me. I've highlighted a few areas I'm going to use.  The title is what we're going to use as the prediction value (highlighted blue), and the other red highlights are categories I want to predict (area, priority, and who the case is assigned to).</p>
<p><img src="http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-30-19_42_05-FogBugz1.png" alt="2013-05-30 19_42_05-FogBugz" width="793" height="480" class="aligncenter size-full wp-image-3965" /></p>
<p>For the impatient, full source code of my bayes classifier is available on my <a href="https://github.com/devshorts/Playground/tree/master/MachineLearning/NaiveBayes" target="_blank" rel="noopener noreferrer">github</a>.</p>
<h2>Conditional probability</h2>
<p>Conditional probability describes the probability of an item given you already know something about it.  Formally it's described in the syntax of P(A | B), which is pronounced as "probability of A given B".  A good example is provided for in the machine learning book. Imagine you have 7 marbles.  3 white marbles, and 4 black marbles. Whats the probability of a white marble? It's 3/7.  How about a black marble? It's 4/7.</p>
<p>Now imagine you introduce two buckets: a blue bucket and a red bucket.  In the red bucket, you have 2 white marbles and 2 black marbles. In the blue bucket you have 1 white marble and 2 black marbles.  Whats the probability of getting a white marble from the blue bucket? It's 1/3. There is only one white marble in the blue bucket, and 3 total marbles.  So, <em>P(white marble | blue bucket)</em> is 1/3. </p>
<p><img src="http://onoffswitch.net/wp-content/uploads/2013/05/buckets.jpg" alt="buckets" width="321" height="144" class="aligncenter size-full wp-image-3952" /></p>
<h2>Bayes Formula</h2>
<p>This doesn't really help though.  What you really want is to be able to calculate <em>P(red bucket | white marble)</em>. This is where bayes rule comes into play:</p>
<p><img src="http://onoffswitch.net/wp-content/uploads/2013/05/d92e290c66d423e4798a22a3690cbd31.png" width="216" height="48" class="alignnone" /></p>
<p>This formula describes how items and their conditions relate (marbles and buckets).</p>
<h2>Conditional Independence</h2>
<p>Naive bayes is called naive because it assumes that each occurrence of an item is just as likely as any other.  Getting a white marble isn't dependent on first getting a black marble. To put it another way, the word "delicious" is just as likely to be next to "sandwich" as it is to "stupid".  It's not really the case. In reality "delicious" is much more likely to be next to "sandwich" than "stupid".</p>
<p>The naive portion is important to note, because it allows us to use the following property of conditionally independent data:</p>
<p><img src="http://onoffswitch.net/wp-content/uploads/2013/05/2013-05-31-14_09_56-www.asz_.ymmf_.hu_talata_math_probability_sections_section_2_6.png" alt="Independent product formula" width="354" height="61" class="aligncenter size-full wp-image-3980" /></p>
<p>What this formula means is that the probability of one thing AND another thing is the probability of each multiplied together. This applies to us since if the text is composed of words, and words are conditionally independent, then we can use the above property to determine the probability of text. In other words, you can expand <em>P(text | spam)</em> to be </p>
<p>```
<br />
text = word1 ∪ word2 ∪ word3 ∪ ... ∪ wordN</p>
<p>P(text | spam) = P(word1 | spam)*P(word2 | spam)...*P(wordN | spam)<br />

```</p>
<p>The probability of the entire text is the probability of each word multiplied together.</p>
<h2>Naive Bayes Algorithm Training</h2>
<p>The goal of training the classifier is to find out what is the probability of a word in a particular class. If we're going to classify the assigned user based on case text, then we need to know what is the probability of a word when the assigned user is "<em>anton kropp</em>", and what is the probability when it is "<em>other developer</em>".</p>
<p>To train the classifier, you need to have some documents whose class you know. For my example, I need to have a bunch of cases already made who are properly assigned.  Using the training documents you first need to find out what are all the available words in the space.  For this I tokenized the case titles into words without any special tokens (?, !, -, <, >, etc) and I removed <a href="http://www.webconfs.com/stop-words.php" target="_blank" rel="noopener noreferrer">commmon stop words</a>.  The word space is the unique set of all the words in the training documents.</p>
<p>The next idea is that you treat this word space as a vector.  If I am training the data on two cases with titles "<em>this is broken</em>" and "<em>users don't work</em>",  then the vector will be:</p>
<p>```
<br />
[this | is | broken | users | dont | work]<br />

```</p>
<p>Where the word "this" is in the 0 position, "is" is in position 1, etc.  Next what we need to do is correlate each document's word occurrence to the training word space vector.  If a word exists it'll get incremented in the right position.  For example, with title "<em>this is broken</em>", it's vector will look like</p>
<p>```
<br />
[1 1 1 0 0 0]<br />

```</p>
<p>Indicating it had 1 occurrence of "<em>this</em>", 1 of "<em>is</em>", and 1 of "<em>broken</em>", but zero occurrences of "<em>users</em>", "<em>dont</em>" and "<em>work</em>".  The second title will have</p>
<p>```
<br />
[0 0 0 1 1 1]<br />

```</p>
<p>Next we need to do is go through every class (the assigned users, for example, such as "<em>anton kropp</em>", and "<em>other developer</em>") and divide the count of each word by the total number of words found. This gives you the probability that a certain word occurs in a certain class.</p>
<h2>Work through an example</h2>
<p>So lets say I have a document class that represents what I'm using to train my data. The document class is generic, it doesn't really care what it's representing.</p>
<p>```csharp
<br />
public class Class<br />
{<br />
    public string Name { get; set; }<br />
}</p>
<p>public class Document<br />
{<br />
    public List&lt;String&gt; Words { get; set; }</p>
<p>    public Class Class { get; set; }<br />
}<br />

```</p>
<p>In my example a document is the case. The words represent the tokenized case title, and the class is the thing I am classifying ("<em>anton kropp</em>" or "<em>other developer</em>").</p>
<p>To build the unique set of vocab in the training space is easy</p>
<p>```csharp
<br />
/// &lt;summary&gt;<br />
/// Create a unique set of words in the vocab space<br />
/// &lt;/summary&gt;<br />
/// &lt;param name=&quot;documents&quot;&gt;&lt;/param&gt;<br />
/// &lt;returns&gt;&lt;/returns&gt;<br />
public static List&lt;string&gt; Vocab(IEnumerable&lt;Document&gt; documents)<br />
{<br />
    return new HashSet&lt;string&gt;(documents.SelectMany(doc =&gt; doc.Words)).ToList();<br />
}<br />

```</p>
<p>Now to train the data.  We group the documents by class, then create a vector representing the occurance of each word in the space for all the documents in that class. I'm initializing the first vector to use all 1's instead of 0's since we are going to multiply the probability of each word together. If any probability is zero (i.e a word wasn't found), then we'll get a zero for the probability, which isn't really true.  </p>
<p>I'm using the <a href="http://mathnetnumerics.codeplex.com/" target="_blank" rel="noopener noreferrer">Math.Net numerics</a> vector library to do the vector processing.</p>
<p>```csharp
<br />
public static TrainedData TrainBayes(List&lt;Document&gt; trainingDocuments)<br />
{<br />
    var vocab = VocabBuilder.Vocab(trainingDocuments);</p>
<p>    var classes = trainingDocuments.GroupBy(doc =&gt; doc.Class.Name).ToList();</p>
<p>    var classProbabilities = new List&lt;ClassProbability&gt;();</p>
<p>    foreach (var @class in classes)<br />
    {<br />
        Vector&lt;double&gt; countPerWordInVocabSpace = DenseVector.Create(vocab.Count, i =&gt; 1);</p>
<p>        countPerWordInVocabSpace = @class.Select(doc =&gt; doc.VocabListVector(vocab))<br />
                                         .Aggregate(countPerWordInVocabSpace, (current, docVocabVector) =&gt; current.Add(docVocabVector));</p>
<p>        var totalWordsFound = 2 + countPerWordInVocabSpace.Sum();</p>
<p>        var probablityVector = DenseVector.OfEnumerable(countPerWordInVocabSpace.Select(i =&gt; Math.Log(i/totalWordsFound)));</p>
<p>        // create an easy to read list of words and its probablity<br />
        var probabilityPerWord = probablityVector.Zip(vocab, (occurence, word) =&gt; new WordProbablity { Probability = occurence, Word = word })<br />
                                                 .ToList();</p>
<p>        var probabilityOfClass = 1.0 / classes.Count();</p>
<p>        classProbabilities.Add(new ClassProbability<br />
        {<br />
            Class = @class.First().Class,<br />
            ProbablityOfClass = probabilityOfClass,<br />
            ProbablitiesList = probabilityPerWord,<br />
            ProbabilityVector = probablityVector<br />
        });<br />
    }</p>
<p>    return new TrainedData<br />
    {<br />
        Probabilities = classProbabilities,<br />
        Vocab = vocab<br />
    };<br />
}<br />

```</p>
<p>You may notice the logarithm stuff going on. That's to prevent <a href="http://stackoverflow.com/a/9342513/310196" target="_blank" rel="noopener noreferrer">numeric underflow</a>. Since probabilities will be small decimals, multiplying them all will make them smaller. By taking the logarithm we can maintain the same relative shape of the function, but they get shifted into a different number space. Now multiplying them won't give us underflow.</p>
<h2>Classifying</h2>
<p>To classify, we need to multiply the vocab vector of a document by each classes probability vector and add the probabilities up.  Whatever class gives us the highest probability is the class we predict we are</p>
<p>```csharp
<br />
public static Class Classify(Document document, TrainedData trainedData)<br />
{<br />
    var vocabVector = document.VocabListVector(trainedData.Vocab);</p>
<p>    var highestProbability = double.MinValue;</p>
<p>    Class classified = null;</p>
<p>    foreach (var @class in trainedData.Probabilities)<br />
    {<br />
        var probablityOfWordsInClass = vocabVector.PointwiseMultiply(@class.ProbabilityVector).Sum();</p>
<p>        var probablity = probablityOfWordsInClass + Math.Log(@class.ProbablityOfClass);</p>
<p>        if (probablity &gt; highestProbability)<br />
        {<br />
            highestProbability = probablity;</p>
<p>            classified = @class.Class;<br />
        }<br />
    }</p>
<p>    return classified;<br />
}<br />

```</p>
<h2>Success rate</h2>
<p>The next step is to test the success rate.  If we have a bunch of documents that are already categorized we can run them back through the classifier.  At that point all that's required is to see if the classifier classifies a document properly or not.  The success rate is important, because if we have a low success rate of classifying documents then either the training set was poor, or the classifier needs other tuning.  Without testing the success rate you can't know how accurate your predictions would be. </p>
<h2>Back to fogubgz</h2>
<p>Remember, I want to see if I can auto triage cases.  To do that I need to train the bayes classifier with some known data.  Thankfully, I have over 50,000 cases to use as a my training set.  Fogbugz makes it easy to pull data back since they have an exposed <a href="http://www.fogcreek.com/fogbugz/docs/70/topics/advanced/api.html" target="_blank" rel="noopener noreferrer">web api</a> you can hook into that returns back neatly formatted XML. First log in:</p>
<p>```
<br />
curl &quot;http://fogbugz.com/api.asp?cmd=logon&amp;email=myEmail@company.com&amp;password=naivebayes&quot;<br />

```</p>
<p>Which gives you a token as a response</p>
<p>```
<br />
&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;<br />
&lt;response&gt;<br />
    &lt;token&gt;<br />
       &lt;![CDATA[5lrnq9c34cjg6h01flg3coe5etj2gg]]&gt;<br />
    &lt;/token&gt;<br />
&lt;/response&gt;<br />

```</p>
<p>Next it's easy to query for a bunch of raw cases and pipe it to some file</p>
<p>```
<br />
curl &quot;http://fogbugz.com/api.asp?cmd=search&amp;q=openedby:anton&amp;cols=sTitle,sProject,sArea,sPersonAssignedTo,sPriority,events,ixPersonOpenedBy,ixPersonResolvedBy&amp;max=1500&amp;token=5lrnq9c34cjg6h01flg3coe5etj2gg&quot; &gt; anton.xml<br />

```</p>
<p>This pulls back 1500 cases that I opened with the colums of title, project, area, assigned to, priority, all the edit events, who opened it, and who resolved it.  I chose 1500 only to not have to wait to pull back all 50,000+ cases (though I probably could have).  Also I am limiting my searches to individual users who have opened cases. By doing this I may be able to get some insight in what kind of cases people make. Maybe cases I make are more likely to be assigned to myself. By limiting my training set to a specific category like this I'm inherently creating a pivot.</p>
<p>After parsing the xml, all you  need to do is transform your fogbugz case to a generic document (that the naive bayes classifier I wrote wants). <code>ToDoc</code> lets me adjust what is the class (in this scenario who the case is assigned to) and what is the predictor text (the case title).  Generalizing the transform lets me reapply the same runner for different combinations of input.</p>
<p>```csharp
<br />
public void TestParser(string path)<br />
{<br />
    var parser = new FogbugzXmlParser();</p>
<p>    var cases = parser.GetCases(path);</p>
<p>    Func&lt;Case, Document&gt; caseTransform =<br />
        i =&gt;<br />
        i.ToDoc(@case =&gt; @case.AssignedTo,<br />
                @case =&gt; @case.Title);</p>
<p>    ProcessCases(cases, caseTransform);<br />
}</p>
<p>private void ProcessCases(List&lt;Case&gt; cases, Func&lt;Case, Document&gt; caseTransform)<br />
{<br />
    var total = cases.Count();</p>
<p>    var trainingSet = cases.Take((int)(total * (3.0 / 4))).Select(caseTransform).ToList();</p>
<p>    var validationSet = cases.Skip((int)(total * (3.0 / 4))).Select(caseTransform).ToList();</p>
<p>    var trainedData = NaiveBayes.TrainBayes(trainingSet);</p>
<p>    var successRate = 0.0;</p>
<p>    foreach (var @case in validationSet)<br />
    {<br />
        if (NaiveBayes.Classify(@case, trainedData).Name == @case.Class.Name)<br />
        {<br />
            successRate++;<br />
        }<br />
    }</p>
<p>    successRate = successRate / validationSet.Count();</p>
<p>    foreach (var type in trainedData.Probabilities)<br />
    {<br />
        Console.WriteLine(type.Class.Name);<br />
        Console.WriteLine(&quot;--------------------
");

type.Top(10).ForEach(i =\> Console.WriteLine("[{1:0.00}] {0}", i.Word, i.Probability));

Console.WriteLine();  
 Console.WriteLine();  
 }

Console.WriteLine("Prediction success rate is {0:0.00}%", successRate \* 100);  
}  

```

## Conclusion

While I can't publish our internal bug information, I found that the classifier ranged in success from as low as 30% to as high as 80% depending on which user I pulled data back from. Personally, cases I opened had a 72.27% success rate classifying their assignment based soley on case title. That's not too bad! It'd be neat to tie this classifier into a fogbugz plugin and generate a first pass auto triage report for our triager to use. When they've reassigned cases we can re-train the set periodically and hopefully get better results.

Full source including bayes classifier and fogbugz parsing available [here](https://github.com/devshorts/Playground/tree/master/MachineLearning/NaiveBayes).

