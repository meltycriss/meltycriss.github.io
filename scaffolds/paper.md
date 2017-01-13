---
title: 论文笔记《Reciprocal Velocity Obstacles for Real-Time Multi-Agent Navigation》
date: {{ date }}
tags: 
  - s2vt_captioner
description:
  - s2vt_captioner.py源码阅读，是LSTM进行test的python脚本
categories:
  - caffe
---

## 简介

s2vt_captioner.py是使用之前训练出来的s2vt model进行caption的脚本。整体逻辑跟训练的时候差不多，但还是有一些区别，比如

- decode时候LSTM2的x不再是gt，而是上一个unit生成的单词
- 使用了python的接口调用caffe

下面按照脚本的核心函数调用链来展开。

## 核心函数调用链

- **main:** 划分为vid chunks
	* **run\_pred\_iters:** 划分为一个个vid
		- **encode\_video\_frames:** 使用vid的feats进行encode
		- **run\_pred\_iter:** decode一个vid，返回captions（有可能多个）
			* **predict\_image\_caption:** 根据caption策略选择对应的caption方式
				- **predict\_image\_caption\_beam\_search:** 使用beam search的方式进行caption
					* **predict\_single\_word:** 生成下一个单词

## main

将任务划分为一个个chunk，交给run\_pred\_iters完成任务

```py
    fsg = fc7FrameSequenceGenerator(filenames, BUFFER_SIZE,
          vocab_file, max_words=MAX_WORDS, align=aligned, shuffle=False,
          pad=aligned, truncate=aligned)
    video_gt_pairs = all_video_gt_pairs(fsg)

```

- **video\_gt\_pairs:** dict{vid: list[gt\_sents]}
	* 假若没有给gt的话，list[gt\_sents]为空


```py
    outputs = run_pred_iters(lstm_net, chunk, video_gt_pairs,
                  fsg, strategies=STRATEGIES, display_vocab=vocab_list)
      
```

- **outputs:** dict{vid: output_batch}
	* 每个vid有可能有多于一个caption，所以是output_batch
- **output_batch:** list[dict{caption, prob, gt, source}]
	* **caption:** 预测的句子，list[word1, word2, word3, ...]
	* **prob:** 句子各单词发生的概率，list[prob\_word1, prob\_word2, prob\_word3, ...]
	* **source:** 搜索句子的策略 e.g. sample / beam

```py
    text_out_types = to_text_output(outputs, vocab_list)

```

- **text\_output\_types:** dict{source\_type: list[output string]}
	* **source\_type:** 搜索句子的策略 e.g. sample / beam

## run\_pred\_iters

将chunk进一步划分为一个个vid，交给run\_pre\_iter完成任务

```py
    # get fc7 feature for the video
    video_features = video_to_descriptor(video_id, fsg)

```

video\_features: list[1 \* FEAT\_DIM]

```py
    # run lstm on all the frames of video before predicting
    encode_video_frames(pred_net, video_features)
    outputs[video_id] = \
        run_pred_iter(pred_net, pad_img_feature, display_vocab, strategies=strategies)

```

caption的流程：

1. 将frame_feats encode到LSTM
2. 从encode好的LSTM中decode出caption

## encode\_video\_frames

```py
    net.forward(frames_fc7=image_features, cont_sentence=cont, input_sentence=data_en,
       stage_indicator=stage_ind)

```

将数据格式匹配输入，将frame\_feats依次传入LSTM，并设置input\_sentence为0

## run\_pred\_iter

```py
    captions, probs = predict_image_caption(net, pad_image_feature, vocab_list, strategy=strategy)
```

对每种strategy都调用predict\_image\_caption来进行caption

## predict\_image\_caption

根据strategy选择对应的image\_caption函数进行caption

## predict\_image\_caption\_beam\_search

使用beam\_search的方式生成句子。所谓的beam\_search实际上为启发式的BFS，使用了贪心的思想：即每次搜索完下一层后，只在下一层中取beam\_size个结点进一步展开，而其余的结点则不要

```py
  beam_size = 1	#每层保留结点数
  beams = [[]]	#beams: list[sentence], sentence: list[word]
  beams_complete = 0	#当前层有多少beam时已经eos了
  beam_probs = [[]]	#粒度为单词，对应beams中每个单词出现的概率
  beam_log_probs = [0.]	#粒度为beam(i.e. 句子)，对应beams中每个beam出现的概率(log w1*w2*w3...)
  current_input_word = 0  # first input is EOS
  while beams_complete < len(beams):
    # expansions: append a new word to current beams
    expansions = []	#记录了下一层单词的信息

	  #每一个单词的信息如下
      # extension : the new word
      exp = {'prefix_beam_index': beam_index, 'extension': [ind],
             'prob_extension': [prob], 'log_prob': extended_beam_log_prob}
      expansions.append(exp)

      #prefix_beam_index: 这个单词是由哪条beam生成出来的
      #extension: 这个单词在字典中的index
      #prob_extension: 产生的是这个单词的概率
      #log_prob: 加上这个单词后，beam的概率log w1*w2*w3...*w_extension

```

了解了数据结构后，先看内层循环

```py
    #       0       p_w1, p_w2...  w1, w2...
    for beam_index, beam_log_prob, beam in \
        zip(range(len(beams)), beam_log_probs, beams):
      if beam:
        previous_word = beam[-1]
        if len(beam) >= max_length or previous_word == 0:
          exp = {'prefix_beam_index': beam_index, 'extension': [],
                 'prob_extension': [], 'log_prob': beam_log_prob}
          expansions.append(exp)
          # Don't expand this beam; it was already ended with an EOS,
          # or is the max length.
          continue
      else:
        previous_word = 0  # EOS is first word
      if beam_size == 1:
        probs = predict_single_word(net, pad_img_feature, previous_word)
      else:
        probs = predict_single_word_from_all_previous(net, pad_img_feature, beam)
      assert len(probs.shape) == 1
      assert probs.shape[0] == len(vocab_list)
      # index of top beam_size prob words
      expansion_inds = probs.argsort()[-beam_size:]
      for ind in expansion_inds:
        prob = probs[ind]
        extended_beam_log_prob = beam_log_prob + math.log(prob)
        # extension : the new word
        exp = {'prefix_beam_index': beam_index, 'extension': [ind],
               'prob_extension': [prob], 'log_prob': extended_beam_log_prob}
        expansions.append(exp)

```

将每条beam生成的单词中，概率最高的beam\_size个单词保存到expansions中。内层循环结束后，expansions中含有len(beams) * beam\_size个单词的信息。接下来看外层循环

```py
  while beams_complete < len(beams):
    # expansions: append a new word to current beams
    expansions = []

  	#内层循环

    # Sort expansions in decreasing order of probabilitf.
    expansions.sort(key=lambda expansion: -1 * expansion['log_prob'])
    # only reserve beam_size number of node in each BFS layer
    expansions = expansions[:beam_size]
    new_beams = \
        [beams[e['prefix_beam_index']] + e['extension'] for e in expansions]
    new_beam_probs = \
        [beam_probs[e['prefix_beam_index']] + e['prob_extension'] for e in expansions]
    beam_log_probs = [e['log_prob'] for e in expansions]
    beams_complete = 0
    for beam in new_beams:
      if beam[-1] == 0 or len(beam) >= max_length: beams_complete += 1
    beams, beam_probs = new_beams, new_beam_probs

```

保留expansions中概率最高的beam\_size个单词，并用这bean\_size个单词扩展beams，得到size为beam\_size的new_beams。一直循环，直到beams中所有beam都结束

## predict\_single\_word

同理encode\_video\_frame，不过这里以pad作为frames_fc7的输入

## 关于caffemodel参数不匹配的问题

可以看到

- 在使用s2vt\_captioner.py进行**test**的时候，网络的模型是**没有展开的**，然后通过将每次的output作为下次的input来传递c和h
- 然而，在**train**的时候，可以看到网络的模型是将LSTM**展开的**
- 显然，展开的LSTM的参数会比不展开的**参数要多得多**
- 那为什么还能用**train出来的caffemodel给test用呢？**

原因就在于，train的时候，展开的各部分实际上是share同一份参数的。而实现参数共享的trick就在我一开始想不明白有什么用的`add_param()->set_name()`

```cpp
  //lstm_layer.cpp

  // Add layer to transform all timesteps of x to the hidden state dimension.
  //     W_xc_x = W_xc * x + b_c
  //{
  //  LayerParameter* x_transform_param = net_param->add_layer();
  //  x_transform_param->CopyFrom(biased_hidden_param);
  //  x_transform_param->set_name("x_transform");
    x_transform_param->add_param()->set_name("W_xc");
    x_transform_param->add_param()->set_name("b_c");
  //  x_transform_param->add_bottom("x");
  //  x_transform_param->add_top("W_xc_x");
  //}

  // Add layer to compute
  //     W_hc_h_{t-1} := W_hc * h_conted_{t-1}
  //{
  //  LayerParameter* w_param = net_param->add_layer();
  //  w_param->CopyFrom(hidden_param);
  //  w_param->set_name("transform_" + ts);
    w_param->add_param()->set_name("W_hc");
  //  w_param->add_bottom("h_conted_" + tm1s);
  //  w_param->add_top("W_hc_h_" + tm1s);
  //  w_param->mutable_inner_product_param()->set_axis(2);
  //}

```

再从caffe.proto中看这个param参数的含义

```proto
//caffe.proto

message LayerParameter {
  // Specifies training parameters (multipliers on global learning constants,
  // and the name and other settings used for weight sharing).
  repeated ParamSpec param = 6;
}

```

所以说，所有参数都是共享同一份的

## 收获

- 在脚本迭代更新，备份上一份脚本时，**备份命名要有意义**，不要只加个.bak就算了，回去一个周末就忘记了.bak1, .bak2, .bak3是哪个跟哪个了
- debug的时候还需要考虑**环境变量**，比如PYTHONPATH这种，特别是两份相同的代码在两台机子上跑出不一样的结果
