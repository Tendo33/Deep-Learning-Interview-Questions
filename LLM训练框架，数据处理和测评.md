# LLM训练框架，数据处理和测评

## 数据处理

LLMs之所以强大，有很大一部分源自其在超大规模数据集上的训练，使得它们各方面能力超越小模型，这就是Scaling的魔力。通常来说，数据量越大，模型效果通常越佳。

一些数据集如C4、The Pile、The Bigsicence Roots Corpus和OpenWebText，甚至于近期的The RefinedWeb Dataset、RedPajama或者说The Stack，都是通过网络爬取并清洗大量文本以提升训练规模。

然而，超大规模数据集的手动审查和筛选的价格高昂且难以保证质量，导致训练出的模型可能带有训练数据的偏见，也就是效果不够好。

同时，对超大规模数据集进行定量和定性研究本就很难。再加上训练中可能还会涉及到Curriculum Learning，以及一些研究表明Loss spike也和训练时batch内数据分布有很大关系，这些都要求我们对互联网数据本身有很清楚的认识，以及一定程度上指导训练的policy。为处理这些问题，第一步就是需确保用于训练LLMs的数据是高质量的。方法包括但不限于：

- **处理无效数据**
    
    一些无效数据，如意义空泛或模板化的文本（例如HTML代码、Lorem ipsum等）。
    
    甚至于在多语言语料库的构建过程中，从网站提取文本用于语言建模也极具挑战性。但这是我们必然要做到的，因为NTP(Next Token Prediction)的方式注定训练模型使用的数据本身就是真实语言世界很好的映射。
    
    数据清洗工具，如justext、trafilatura等，能有效剔除HTML模板文本，同时在减少噪音（提高精度）与保留所有有效部分（提高召回率）之间取得平衡。
    
    另外一点是，处理网页语料库中无效数据的有效方法之一是利用元数据进行筛选。例如，OpenAI在构建GPT-2用的WebText语料库时，抓取了reddit上点赞数至少为3的所有外部链接，这种启发式方法有助于减少数据集中的噪音，同时确保数据质量。
    
- **Document Length Considerations**
    
    一方面，考虑到NTP，从语料库中移除非常短的文档（包含少于约100个标记的文本）可以帮助通过创建连续的文本来建模文本中的依赖关系，从而去除噪音。另一方面，由于大多数语言模型如今都基于Transformer架构，对非常大的文档进行预处理并将其分成所需长度的连续片段是很有用的。
    
- **Machine Generated Text**
    
    训练语言模型的目标之一是捕捉人类语言的分布。然而，网络爬取的数据集包含大量机器生成的文本，例如现有语言模型生成的文本、OCR文本和机器翻译文本。
    
    例如，来自http://patents.google.com的数据构成了C4语料库的大部分。该语料库使用机器翻译将来自世界各地专利机构的专利翻译成英语。此外，网络语料库中的数据还包含来自扫描书籍和文档的OCR生成文本。
    
    OCR系统并不完美，因此生成的文本与自然英语的分布不同（通常OCR系统会在拼写错误和完全遗漏单词等方面产生可预测的错误）——这点很重要，也很难搞，pdf扫描文档怎么去做还真挺头疼的。虽然很难识别机器生成的文本，但有一些工具，如ctrl-detector，可用于识别和检测机器生成的文本。在为语言建模预处理语料库时，重要的是对语料库中机器生成文本的存在进行表征和记录。
    
- **去重和精确去重**
    
    从互联网上爬取原始文本创建的数据集往往会导致相同的序列被多次重复出现。例如，在论文《Deduplicating Training Data Makes Language Models Better》中，作者发现在C4数据集中，一个50个单词的序列被重复出现了60000次。
    
    事实上，在去重的数据集上训练模型速度更快，并且不太容易导致记忆效应——很不好。最近，研究人员还表明，在重复数据上训练的语言模型容易受到隐私攻击，其中对手从训练模型中生成序列并检测哪些序列来自训练集的记忆。
    
    在论文《Deduplicating Training Data Mitigates Privacy Risks in Language Models》中，作者展示了语言模型重新生成训练序列的速率与序列在训练集中的出现次数超线性相关。例如，一个在训练数据中出现10次的序列平均会比一个只出现一次的序列生成1000倍多。去重可以在不同粒度级别上执行。
    
    从精确匹配去重到模糊去重工具（例如deduplicate-text-datasets和datasketch），可以帮助减少和去除正在处理的语料库中的冗余文本。正如许多研究人员所指出的，需要理解去重过程需要大量计算资源（CPU和RAM），因为网页爬取数据集的大小，因此建议在分布式环境中运行此类计算。
    
- **清洗污染数据**
    
    这部分还挺保受争议的，可能还没有很细致的标准，不少公司也都挺功利的，就不好说。在NLP领域，我们常说的数据清洗，主要指的是训练数据和测试数据的区分和处理。
    
    在大型语言模型的情况下，由于训练和测试数据集都源于互联网，确保二者不发生**交叉**，这个过程可能颇具挑战。大型语言模型的评估通常会用到基准数据，如问答对，如果这些基准数据在训练数据中出现，可能会导致基准性能的高估。
    
    因此，需要进行去污染操作，也就是从训练数据中去除和基准数据集有重叠的部分，保证训练数据集的完整性。OpenAI的研究人员在创建WebText数据集时，就通过剔除所有维基百科内容来实现数据去污染，因为维基百科数据在他们的基准数据集中被广泛使用。另一个案例是EleutherAI的研究人员，他们开发了名为lm-eval harness的软件包，用以实现对基准数据集的去污染。在具体操作中，我们需要关注两类数据污染：
    
    1. 输入与输出污染：这种情况下，预训练语料库中存在与下游任务标签相同的数据。对于语言建模等任务，任务标签就是目标文本。如果目标文本在预训练语料库中出现，模型可能会倾向于复制文本，而非真正解决任务。
    2. 输入污染：这指的是评估样本中并未包含标签的情况，这也可能导致下游任务的性能高估。在进行零样本和少样本评估时，如果预训练数据集中存在与热门基准任务重叠的数据，我们必须重视数据去污染。
- **毒性和偏见控制**
    
    尽管网络语料库具有丰富的多样性，但其中也常常弥漫着毒性和偏见内容。如，《RealToxicityPrompts》一文中作者使用PerspectiveAPI指出，OpenWebText与WebText的内容中分别有2.1%与4.3%存在毒性分数超过50%。
    
    因此，在训练语言模型时，必须警觉并借助PerspectiveAPI等工具筛选掉预训练数据集中的毒性内容，以防止模型表现出偏见或在下游应用中产生有害内容。一种解决策略是过滤掉"bad words"名单中的文本，比如C4的作者们就采用了这种策略。
    
    另一个例子是，PILE数据集的研究者利用spamscanner来对有害内容进行分类。然而，执行此类过滤步骤必须极为谨慎，并需考虑到下游应用，以免过滤器保留下更可能坚持霸权观点的声音。在利用数据进行预训练语言模型之前，对贬损内容和性别/宗教偏见进行深度分析是必要的。
    
- **个人身份信息控制**
    
    在收集大型数据集时，理解与数据集实例相关的法律问题至关重要，尤其是在处理个人身份信息（PII）时，如真实姓名、组织名称、医疗记录、社会安全号码等。
    
    根据不同的应用，对这些信息进行遮蔽或删除在预训练语言模型之前是必要的。像presidio和pii-codex这样的工具提供了检测、分析和处理文本数据中个人身份信息的流程，这些工具能帮助确保数据集中的个人信息得到合理处理，以遵守相关隐私法规并保护用户隐私。
    

---

- **bloom, llama，glm等开源模型的数据来源，配比，以及不足之处**
    
    在谈论各个base model之前，我们得先清楚pretrain的数据来源的分类，大致如下
    
    - 自然语言
    - 书籍/文章
    - Book
    - 网页
    - Arxiv
    - Wikipedia
    - C
    - OSCAR
    - Common Crawl
    - StackExchange
    - 其他数据集
    - 编程语言
    - 网页
    - Github（Google BigQuery）
    - StackOverflow
    - 其他数据集
    
    注意这里所谓的数据集就是有人替你爬好了而已。其实语料的来源并不那么重要，因为总的来说各个模型用的源语料基本上都是这些（多样性较易达成） 重点在于怎么去清洗。
    

---

- **除了loss之外，如何在训练过程中监控模型能力？**
    
    [大模型RLHF的trick](https://mp.weixin.qq.com/s?__biz=MzIwNDY1NTU5Mg==&mid=2247486087&idx=1&sn=948eb9c0077e6eb360c6c61bf9ef1bca&chksm=973d9400a04a1d161636fa2e789fe8a7005efbcd54c9d19f9f0146be799fdc91278280498470&scene=21#wechat_redirect)
    
- **如果想全面的评测模型能力，有哪些维度以及数据集？评测指标等评测中比较重要的部分要了解。**
    
    评测的核心逻辑是什么？一个模型的评测需要和模型被期望获得的能力紧紧相关.
    
    - 评价base model
        
        考虑到一般认为pretrain过程是模型获得99%能力的阶段，我们需要关注的是具有一般通用的核心能力，尤其是一些具有划分度的推理/泛化能力.
        
        - 找有区分度，能体现核心能力的数据集，下面可以参考：
        - 英文知识 — MMLU
        - 中文知识 — C-Eval
        - 推理 — GSM8k / BBH
        - 代码 — HumanEval / MBPP
        - 数学 — MATH
        - 用体现这些能力的数据集，来做automatic eval：
            - 可以参考chain-of-thought-hub，但是评测中的难点就在于不同prompt的设计对于不同模型的能力挖掘是不一样的，所以可能相对公平的方法就是用实际业务可能的prompt来eval（除非是想公开自己研发的llm的评测board，那自然是需要做一些prompt engineering的方法来努力提点）
    - 评价chat bot
        
        考量提升用户的体验的相关能力，比如说3H(helpful,honesty,harmless)等等
        
    - 领域微调的model
        
        那自然有行业的基准去评测。因此，说在前面的是，刷榜是不必要的，因为我们更多地是考量模型的能力，只不过能力的衡量是体现在这些预设的指标上的。
        
        指标的衡量上，你确保你的模型在这方面的能力equip，你的training strategy works就好。我们的期望当然是所谓的评测能够证明模型的能力强，只是相比通过刷榜来证明，你可能更多关注的是为什么在某些题上面表现不好，是与训练的数据质量有关，还是你的评测本身就没有激发出模型的能力等等（你的模型能力够强，你的评测一定不会差；你的评测效果很好，未必说明你的模型就是ok的）。毕竟刷榜本身也和model的实际应用场景并不契合，是有偏的。