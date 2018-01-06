# Naive Bayes Classification
> 根據分佈機率的學習方法

C_map = arg max P(c|d) (我們想知道： **根據文件，可以將其分到哪一個類別**)
可由貝氏定理推導：
P\(c|d\) = P\(c^d\) / P(d) = **P\(d|c\) * P\(c\) / P(d)**
> ( c: 類別, d: 文件 )
P\(c\)，Prior probability
P\(d|c\)，Posterior probability，因為它反應了我們看到d後，對於分到該類別的信心指數
P\(d\)，因為都大家都共享這個分母，因此可以直接消去 (對分類沒有貢獻)

Naive Bayes又分為兩種
  - Multinomial mode
  關注於文件裏的詞的 **出現頻率**
  - Binomial model (Bernouli model) 
  關注於文件裏的詞 **是否** 出現

P\(c\)很好算，那P\(d|c\)呢？
根據chain rule：
P\(d|c\)
= P(t1|c) * P(t2|t1, c) * P(t3|t2, t1, c) * ... * P(t2|tn, ..., t2, t1, c)
= [ P(t1,c) / P\(c\) ] * [ P(t1,t2,c) / P(t1,c) ] * ... * [P(t1,t2,...,tn,c) / P(t1,t2,...t_n-1,c)] 
= P(t1,t2,...,t_n,c) / P\(c\) = **P(t1,t2,...,tn | c)**

> 因為這樣要考慮的模型太過複雜(M+M^2+M^3+...+M^n，每次字出現的機率還要顧及上一個位置出現的機率)

因此Naive Bayes提出了一個假設 "**conditional independence assumption**"(其他word是否出現，不影響這個字的出現機率)
因此P(d|c) = P(t1|c)P(t2|c)P(t3|c)...P(tn|c)
**(  P(x1=chinese|c) != P(x3=chinese|c)  )**
> 但這樣又有出現新的問題：
(1) 我們無法事先得知文章的長度
(2) 資料的稀疏性

因此Naive Bayes提出了另一個假設 "**positional independence assumption**"(不管在第幾個位置，不影響這個字的出現機率)
這樣P(t1|c) = P(t2|c) = ... = P(tn|c)
但也失去了字裡行間的順序，變為 **bag of words**
e.g. Asus merged with Acer = Acer merged with Asus
> 這也是為什麼"Naive" Bayes的由來
因為它迫於複雜度，要假設兩個"天真的"假設

