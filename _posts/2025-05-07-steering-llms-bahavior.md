---
layout: distill
title: Steering LLMs' Behavior with Concept Activation Vectors
description: Concept activation vectors have been shown to take effects in safety concepts, efficiently and effectively guiding a considerable number of open-source large language models (LLMs) to respond positively to malicious instructions. In this blog, we aim to explore the capability boundaries of concept activation vectors in guiding various behaviors of LLMs through more extensive experiments. Our experiments demonstrate that this reasoning technique can low-costly transfer text styles and improve performance on specific tasks such as code generation. 
date: 2025-05-07
future: true
htmlwidgets: true
hidden: false

# Anonymize when submitting
authors:
  - name: Anonymous

# authors:
#   - name: Albert Einstein
#     url: "https://en.wikipedia.org/wiki/Albert_Einstein"
#     affiliations:
#       name: IAS, Princeton
#   - name: Boris Podolsky
#     url: "https://en.wikipedia.org/wiki/Boris_Podolsky"
#     affiliations:
#       name: IAS, Princeton
#   - name: Nathan Rosen
#     url: "https://en.wikipedia.org/wiki/Nathan_Rosen"
#     affiliations:
#       name: IAS, Princeton

# must be the exact same name as your blogpost
bibliography: 2025-05-07-steering-llms-bahavior.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly. 
#   - please use this format rather than manually creating a markdown table of contents.
toc:
    - name: Preliminaries
    - name: Methodology
    - name: Experiments
        subsections:
        - name: Concept 1: Python (Code)
        - name: Concept 2: Language
        subsections:
        - name: French Concept
        - name: Simplfied/Traditional Chinese Concept
        - name: Arabic Concept
        - name: Areas where CAV Excels and Does Not
    - name: Discussion
        subsections:
        - name: Is PSA-induced CAV the same as IT-induced?
        - name: Can expand to multi-behavior steering?
    - name: Conclusion


# Below is an example of injecting additional post-specific styles.
# This is used in the 'Layouts' section of this post.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
---

Note: please use the table of contents as defined in the front matter rather than the traditional markdown styling.

As a classic interpretation method, concept activation vectors (CAVs) describe the distribution features of activation in neural networks with simple linear classifiers. Some work has successfully transferred this to LLMs, using CAV to steer model behavior. Previous research on the ability of CAV to steer LLMs' behavior has not been systematic enough, limited to safety concepts <d-cite key="Xu2024uncovering"></d-cite> or insufficient hyperparameter tuning <d-cite key="nejadgholi2022improving"></d-cite>. Other methods not using CAV to steer LLMs' behavior, such as simply merging residual streams <d-cite key="turner2023activation"></d-cite>, is even less effective than the algorithm proposed in <d-cite key="Xu2024uncovering"></d-cite>, which can automatically merge multiple layers and determine hyperparameters. This blog aims to explore the boundaries of using CAV to steer model behavior through extensive experiments and observations, fully showcasing its potential and limitations.

Note: Since CAV originated from image neural networks interpretation, its "activation" refers to the embeddings or representations of LLMs. For the sake of coherence, this blog uses "activation" when describing concepts related to CAV and uses "embedding" when describing concepts related to LLMs.

## Preliminaries

For single behavior steering, we use the pipeline outlined in <d-cite key="Xu2024uncovering"></d-cite>. The main difference between using CAV for steering and methods like ActAdd  <d-cite key="turner2023activation"></d-cite> is that the linear classifiers relied upon by CAV can utilize the embedding distribution characteristics of LLMs to the maximum extent, enabling behavior steering at minimal cost of performance damage.

### Overview of CAV Perturbation Algorithm 

1. **Data Collection**: Gather two sets of instructions aimed at carrying out two different tasks, labeling one as the positive and the other as negative. For example, a positive dataset might include instructions like

   $$ \quad\textcolor{red}{\text{Positive}:}\text{How to plant flowers in my garden?} $$

   while the negative dataset might include instructions like

   $$ \quad\textcolor{green}{\text{Negative}:}\text{Comment planter des fleurs dans mon jardin?} $$

   Thus, this two datasets can be used for "French" concept steering. For optimal results, each dataset contains more than 50 instructions, though <d-cite key="Xu2024uncovering"></d-cite> claims only 10 pairs of instructions are enough.

2. **LLM Selection**: Choose a target LLM, such as *LLaMA-3-8B*, known for its better capabilities. Collect the final token embeddings from each layer for instructions in positive and negative datasets respectively.

3. **Classifier Training**: Train a linear classifier on these embeddings with their corresponding labels for each layer. That's to say, we will get $$ N $$ classifiers for steering a single behavior, where $$ N $$ is the total number of transformer layers of the targeted LLM.

4. **Text Generation with Perturbation**: Use the trained classifier parameters to perturb a typical text generation process of the targeted LLM. Due to the transferability disclosed by <d-cite key="Xu2024uncovering"></d-cite>, it may be possible to apply the CAV trained on targeted LLM to other LLMs. However, since we have complete access to all open-source LLMs, it is assumed that the LLM used for training and generation is same by default.

### Perturbation Process Details

Assuming classifiers have been trained for each layer, we perform perturbations sequentially during the generation process for each new token:

- For each layer $$ n $$ , we first evaluate the test accuracy of its corresponding classifier. If the accuracy is low, indicating the concept is not yet fully formed at this layer, we skip perturbation to minimize potential adverse impacts on the LLM’s overall capabilities. Conversely, if the accuracy exceeds a specified threshold $$ P_1 $$, we proceed with the perturbation.
- Let $$\theta$$ be the parameters of classifier $$f$$, and $$e$$ the original layer embedding. The perturbation process then depends on the intended direction of steering. If the input instruction originally targets a task described by the positive label and the goal of steering is to shift towards the negative, we reduce $$f_\theta(e)$$ to achieve a targeted probability $$P_0$$.
- The perturbation is represented by $$ e\leftarrow e+g(\theta,e,P_0) $$, where $$g(\cdot)$$ is a closed-form function. This adjustment across layers modifies the LLM’s “recognition” of the input instruction, leading the output to exhibit characteristics represented by the negative label.

## Methodology

We use the benign instructions provided in representation engineering <d-cite key="zou2023representation"></d-cite> as the training set base, as this dataset includes many normal instructions for LLMs. For each behavior steering case, we use 50 instructions as the training set and another 50 non-overlapping instructions as the test set to calculate the test accuracy of the classifiers. In other words, we extract 4 subsets of 50 instructions from the original dataset, with two assigned to each label for training and testing.

In this blog, our positive instructions are the unmodified version, while the negative instructions have been modified to have steering goal characteristics. This setting is consistent with the example above <d-cite key="Xu2024uncovering"></d-cite>.

To construct the negative dataset, that is, to modify negative instructions into the form with targeted behavior, we summarize three methods:

1. **Complete Replacement**. For fundamentally different tasks, the original instruction can be directly replaced with a completely different instruction that has the target features. For example, in safety tasks, the instructions in the negative dataset are all recollected.

2. **Prefix and Suffix Addition**. For tasks like style transfer, a string describing requirements can be added in the form of a prefix or suffix to the original instruction. For instance, adding `"Answer in Python code please."` is a processing method suitable for "Code" concept. 

   To avoid potential misguidance caused by a single format of additional requirement string, this modification has two random elements - randomly using a prefix or suffix, and randomly selecting one string from several to add to the original instruction.

3. **Instruction Transfer**. For tasks like language or grammar, the original instruction can be directly transferred to the target task. For example, for "French" concept, the original instruction can be translated into the corresponding language.

For example (CR - *Complete Replacement*; PSA - *Prefix and Suffix Addtion*; IT - *Instruction Transfer*):

$$\quad x=\text{How to plant flowers in my garden?}$$

$$\quad\textcolor{blue}{\text{CR}}(x)=\text{How to make American coffee?}$$

$$\quad\textcolor{orange}{\text{PSA}}(x)=\text{How to plant flowers in my garden? Answer in Python please.}$$

$$\quad\textcolor{pink}{\text{IT}}(x)=\text{Comment planter des fleurs dans mon jardin?}$$

The above three operations can be seen as operational primitives, and the results obtained by nesting them can serve as a method for constructing datasets for multi-target behavior steering, which will be described in detail later.

$$\quad\textcolor{orange}{\text{PSA}}(\textcolor{pink}{\text{IT}}(x))=\text{Comment planter des fleurs dans mon jardin? Answer in Python please.}$$

After building the datasets, we perform the CAV perturbation in text generation process, achieving the effect of behavior steering.

## Experiments

Before training classifiers, we applied PCA reduction to evaluate the separability of the embeddings of these instructions. We trained the PCA on the dataset and observed good linear separability. We want to clarify that the intuitive results of PCA are not completely consistent with the actual test accuracy. In cases where positive and negative examples appear to overlap in the PCA results, the classifier's test accuracy may still be very high, even as high as those layers that do not seem to overlap. These special cases will be marked in corresponding subsections.

By default, we use `Llama-3-8B-Instruct` for experiments. Other LLMs may be involved for some concepts, and we will clearly indicate.

### Concept 1: Python (Code)

First, we try the Python concept, which has a negative instruction dataset constructed by PSA (Perfix and Suffix Addition). The test accuracy for the CAV is quite high, above 99% except for layer 0. However, you will see in the PCA results shown below that the early layers seem to have better separability than the later layers. Therefore, the results of PCA can only be auxiliary, and the test accuracy is a better indicator for understanding the effectiveness of CAV.

{% raw %}{% include figure.html path="assets/img/2025-05-07-steering-llms-bahavior/image-20241031233343529.png" class="img-fluid" %}{% endraw %}

After training the Python CAV, we will attempt to steer model behavior with it. We will apply Python CAV to three types of tasks: 

1. Tasks completely unrelated to code, e.g., **how to make American coffee**;
2. Tasks with a certain programming mindset requirement, e.g., **how to solve Hanoi Tower problem**;
3. Complete programming tasks, e.g., **how to calculate the maximum depth of a binary tree**. The example is a classic algorithm problem, and we can see whether the solution is better with steering.

You can try the interactive section below to compare the outputs of the three tasks before and after using Python CAV to steer behavior.

<div class="l-page">
  <iframe src="{{ 'assets/html/2025-05-07-steering-llms-behavior/1.html' | relative_url }}" frameborder='0' scrolling='no' height="100%" width="100%"></iframe>
</div>

Our observation is, for some specific tasks, Python CAV can enable LLMs to provide answers containing Python code, even if the original response doesn't include Python code, or the original instruction is not a coding task at all. However, for coding tasks, there is a lack of broader experiments to demonstrate whether it will be better steering with CAV. Referring to the results of Task 1.3, the response before steering are more comprehensive, while the response after steering seem to be more straightforward.

### Concept 2: Language

Next, we explore language-related concepts, which are considered a more practical steering usage. The experiments of language concepts involves four specific languages—English, French, Chinese (including Simplified and Traditional), and Arabic. The construction of the dataset involves two methods—Prefix and Suffix Addition (PSA) and Instruction Transfer (IT). Due to space limitations, it is not possible to present the results for all combinations pairwise; only some of the most meaningful and interesting content will be discussed below.

#### French Concept

When studying the French concept, we also examine the differences between PSA and IT. When using PSA to induce French CAV, the instructions in both the positive and negative datasets are written in English. When using IT, the instructions in the positive dataset are in English, while the instructions in the negative dataset are translated into French by a translation API. 

We selected three different text generation tasks to test the effects of using PSA and IT to induce the French CAV for behavior steering. Please operate the interactive panel below to view the PCA results and outputs of the two CAVs.

<div class="l-page">
  <iframe src="{{ 'assets/html/2025-05-07-steering-llms-behavior/2.html' | relative_url }}" frameborder='0' scrolling='no' height="100%" width="100%"></iframe>
</div>

Our observations are:

1. Using IT-induced French CAV achieves better effect than PSA-induced one; despite this, their test accuracies are quite high;

2. When steering with PSA-induced CAV, the output often retains more or less English content, while using IT-induced CAV results in much less, although some English content still exists;

3. The optimal $$P_0$$ for steering with the two types of CAV are different, with about 25% for PSA-CAV and about 10% for IT-CAV. This difference seems to show that even we look like to induce the same CAVs, the results are different. This will be discussed further in the later section.

{% raw %}{% include figure.html path="assets/img/2025-05-07-steering-llms-bahavior/image-20241101205150263.png" class="img-fluid" %}{% endraw %}

#### Simplfied/Traditional Chinese Concept

The differences between simplified and traditional Chinese are a very interesting phenomenon. We use IT and translation APIs to construct positive and negative datasets, with the positive dataset translated into simplified Chinese and the negative dataset into traditional Chinese. However, we struggle to train a good CAV on `Llama-3-8B-Instruct` with such datasets, possibly because this model doesn't have good Chinese output capabilities. Therefore, we use `Llama3.1-8B-Chinese-Chat`, a fine-tuned version of `Llama3.1-8B-Instruct` and its original version for this concept.

<div class="l-page">
  <iframe src="{{ 'assets/html/2025-05-07-steering-llms-behavior/3.html' | relative_url }}" frameborder='0' scrolling='no' height="100%" width="100%"></iframe>
</div>

{% raw %}{% include figure.html path="assets/img/2025-05-07-steering-llms-bahavior/image-20241101195212504.png" class="img-fluid" %}{% endraw %}

{% raw %}{% include figure.html path="assets/img/2025-05-07-steering-llms-bahavior/image-20241101202314773.png" class="img-fluid" %}{% endraw %}

Our observations are:

1. `Llama-3-8B-Instruct` answers all three tasks tested with English as the input language in Simplified Chinese, so there was no noticeable change after applying CAV perturbations; `Llama-3.1-8B-Instruct` and `Llama-3.1-8B-Chinese-Chat` are able to respond in Chinese, indicating that the CAV perturbations are effective. The text in the interactive panel above is generated by `Llama-3.1-8B-Chinese-Chat`;

2. The accuracy trends of the CAVs trained on the three mentioned LLMs show a similar pattern across layers, initially decreasing and then increasing. In terms of relative accuracy, `Llama-3-8B-Instruct` is the lowest, while `Llama-3.1-8B-Instruct` is the highest (with some later layers of `Llama-3.1-8B-Chinese-Chat` being even higher). In certain middle layers, the difference between `Llama-3.1-8B-Instruct` and `Llama-3.1-8B-Chinese-Chat` is even greater than the difference between `Llama-3.1-8B-Chinese-Chat` and `Llama-3-8B-Instruct`. The reason for this is currently unclear.

#### Arabic Concept

Compared to French and Chinese, Arabic should be a less common language in Llama-3.1 and is also a low-resource language. How effective is the CAV extraction and behavior steering for this low-resource language? We also used PSA and IT methods along with the Arabic translation API to build datasets. Try the interactive panel below to see the steering results.

<div class="l-page">
  <iframe src="{{ 'assets/html/2025-05-07-steering-llms-behavior/5.html' | relative_url }}" frameborder='0' scrolling='no' height="100%" width="100%"></iframe>
</div>

Our observations are:

1. Basically, it is possible to give instructions in English and let the LLM respond in Arabic without any additional prompts. However, this steering is very unstable, which is why we present the results of steering under different $$P_0$$. Even in most of the successful responses that are answered in Arabic, there are still a small number of characters in English or other languages. We are not proficient in Arabic, so we use GPT-4o to translate these Arabic passages, revealing that the content appropriately completed the given instruction.
2. Interestingly, under both the PSA and IT, an intriguing phenomenon occurs. In some cases, the LLM responds extensively in languages other than Arabic and English, including Indonesian, Spanish, Italian and Russian. Moreover, it is not the case that a smaller $$P_0$$ guarantees a higher likelihood of Arabic responses; we found that in some cases, responses in Arabic occur when $$P_0$$ is larger, while responses in other non-English languages occur when $$P_0$$ is smaller. This may be due to the concepts in low-resource languages being too close to each other or not presenting linearly, which causes the CAV based on linear models to not describe them well.
3. Even on LLMs specifically fine-tuned for Arabic, such as `MohamedRashad/Arabic-Orpo-Llama-3-8B-Instruct`, the above phenomenon remains.

### Areas where CAV Excels and Does Not

From the results of the two concepts above, it seems that CAV is quite good at modifying the style of the entire generated content. We have attempted more style concepts based on PSA, such as telling jokes, being more creative, childish, fairy tale-like, etc. The results show that CAV performs well in steering these concepts. Please use the interactive panel below to view the results of various style transfers on three specific instructions.

<div class="l-page">
  <iframe src="{{ 'assets/html/2025-05-07-steering-llms-behavior/4.html' | relative_url }}" frameborder='0' scrolling='no' height="100%" width="100%"></iframe>
</div>

In addition to these concepts, we also tried many other concepts, but fail to demonstrate such effects. These PSA-based CAVs all had quite high test accuracy, but after multiple attempts with different values of $$P_0$$, we could never produce significantly steered responses. These concepts include:

1. "You are a character from Genshin Impact (Authors' note: a globally released RPG game)"

2. "Your name is Kimi, developed by Anthropic"

3. "Remember 9.11 < 9.8"

4. "Every word in your response must be repeated twice"

5. "Your answer should include a special symbol ^_^"

6. "Act as if you forget anything about *the Bible*"

However, some interesting phenomena can also be found from these failures:

1. Since Llama-3.1 itself cannot provide consistent responses to comparison instructions like x.11 and x.8 when x varies, it is impossible to determine whether the results of our steering are reasonable. We find that $$P_0=1\%$$ and $$P_0=99\%$$ let the model to give completely opposite results, which somewhat aligns with our expectations. Additionally, if $$P_0$$ is not set to such extreme values, it appears to have no steering effect.

2. The two requirements "Every word in your response must be repeated twice" and "Your answer should include a special symbol ^_^" failed when we set the CAVs in the desired direction without explicitly asking for them in the instructions. But this doesn't mean that the extracted CAVs are useless. If we explicitly add these two requirements in the instructions and set the CAVs in the direction of not doing so, we can indeed disable the effect of the added prompts. To prove that this is not caused by other reasons, we conduct an ablation study using CAVs trained on other concepts, and the results confirmed that the effect of disabling is indeed caused by the CAV trained on corresponding concept.

In addition, we also find that the behavior steering effect of CAV performs better in longer responses, and the first few tokens of the response seem to retain the characteristics before steering, which indicates that the steering caused by CAV requires a few tokens to adapt.

## Discussion

### Is PSA-induced CAV the same as IT-induced?

Through the experiments described above, we observe that both settings could effectively extract a French concept and achieve forward and reverse behavior steering. This raised an interesting question regarding the uniqueness of the French concept. The success in both directions suggests a relationship akin to a sufficient and necessary condition, implying that the French concept within LLMs may be unique. To explore this further, we extended the experiment settings as follows:

- **Dataset A**: Normal instructions.
  - How to plant flowers in my garden?
  
- **Dataset B**: Directly use French. (using IT)
  - Comment planter des fleurs dans mon jardin?
  
- **Dataset C**: Request using French. (using PSA)
  - How to plant flowers in my garden? Answer the question in French please.
  
- **Dataset D**: Request using English. (using PSA+IT)
  - Comment planter des fleurs dans mon jardin? Répondre en anglais.
  
- **Dataset E**: Unrelated postive dataset.

- **Dataset F**: Unrelated negative dataset.

We trained CAVs using the following pairs of datasets:

- Pair #1: A-B
- Pair #2: A-C
- Pair #3: B-D
- Pair #4: C-D
- Pair #5: E-F (baseline)

The classifiers trained on these five pairs exhibited good test set accuracy. To further understand their behavior, we examined the cosine similarity between the parameters of these classifiers:

{% raw %}{% include figure.html path="assets/img/2025-05-07-steering-llms-bahavior/bzvvxffran2k356edpf6.png" class="img-fluid" %}{% endraw %}

Our observations are:

- Despite similar steering results, the classifiers trained seem to be rather different (see the cosine similarity between Pair #1 and #2/#3), suggesting substantial differences in their internal representations.
- The cosine similarity between Pair #2 and #3 is relatively high, which aligns with our intuition since these pairs involve similar types of instructions (requests in English or French).
- The cosine similarity between baseline and others are close to 0, demonstrating the effectiveness of the comparison and validating that the steering methods were indeed capturing meaningful differences related to the targeted language concept.

### Can expand to multi-behavior steering?

Based on the methodology of single-behavior steering, there are two approaches to using CAV for multi-behavior manipulation. Assume there are three target behaviors A, B, and C.

1. **Train the CAVs for A, B, and C separately**, and apply these CAVs to perturb the embedding in a certain order or in a limited loop.
2. Modify the negative dataset to have instructions that simultaneously have the characteristics of behaviors A, B, and C, and **train only one CAV**.

There has already been some preliminary research on this <d-cite key="scalena2024multiproperty"></d-cite>. The first method needs consider the orthogonal nature between concepts; otherwise, the probabilities will oscillate at each perturbation step and fail to converge. The second method is straightforward, requiring only the use of PSA, IT, or their combinations, but whether CAV can represent such complex content and whether it will increase the damage to text quality needs to be tested.

## Conclusion

In this blog, we explored the breadth and boundaries of using CAV for LLM behavior steering. Using CAV for steering, PSA and IT are good ways to construct datasets, allowing for easy export of the corresponding CAV. CAV-based steering is more suitable for tasks that require the transfer of overall text style, including the language used in the text, and has the ability to disable some brief system prompt requirements, but cannot perform more complex cognitive demands. Research on CAV steering can more effectively promote the exploration of explainable AI and low-cost text style transfer generation.