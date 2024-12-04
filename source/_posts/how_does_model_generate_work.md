---
title: How does model.generate work in transformers? (from llama's perspective)
time: 2024/11/30
tags:
    - LLM
    - AI
category: 技术学习与分享
---


`transformers` is a well-known python package that wraps various model's implementation. It provides an easy way to implement/modify/create various models. 

A usual implementation for causal llm look like below:

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")

inputs = tokenizer("Hello world!", return_tensors="pt")
outputs = model.generate(**inputs,...)
```

outputs would contain a tensor for tokenizer to decode. 

This article is to have an insight into how this `model.generate` works. To understand it, we need to look into the source code of transformers. We use `transformers==4.44`, and as transformers is being developed so actively, some of the code here may be outdated. 

We focus on how the class GenerationMixin use the `forward` function in this article (How this 'forward' function work in LlamaForCausalLM is another topic). In this topic, we only need to know that LlamaForCausalLM.forward takes in some input_ids and other bunches of parameters, and return something called logits to its caller. 

Here's an overview when you call the function `model.generate`
![alt text](attachments/image.png)

![alt text](attachments/image-1.png)

Let's begin with understanding this `input` argument in this `generate` function. Input is an object of a class called `BatchEncoding`, and it contains bunches of tensors like input_ids, attention_mask, etc. But that would be too much to discuss here. Anyway, `generate` extract data from this `input` with a member function called `GenertionMixin._prepare_model_inputs`

```python
# 3. Define model inputs
inputs_tensor, model_input_name, model_kwargs = self._prepare_model_inputs(

	inputs, generation_config.bos_token_id, model_kwargs

)
```
After *defining other model kwargs*, we can generate input_ids, which will be passed to the `forward` function of Llama. 

```python
# 5. Prepare `input_ids` which will be used for auto-regressive generation

if self.config.is_encoder_decoder:

	input_ids, model_kwargs = self._prepare_decoder_input_ids_for_generation(

		batch_size=batch_size,

		model_input_name=model_input_name,

		model_kwargs=model_kwargs,

		decoder_start_token_id=generation_config._decoder_start_token_tensor,

		device=inputs_tensor.device,

	)

else:

	input_ids = inputs_tensor if model_input_name == "input_ids" else model_kwargs.pop("input_ids")
```

Then the code did some work about *caching and generation constraints*. 

After that, the code need to determine the `generation_mode` of the model, create `prepared_logits_processor` and  `prepared_stopping_criteria` for future use. 

```python
# 7. determine generation mode

generation_mode = generation_config.get_generation_mode(assistant_model)

if streamer is not None and (generation_config.num_beams > 1):
	raise ValueError(
		"`streamer` cannot be used with beam search (yet!). Make sure that `num_beams` is set to 1."
	)

if not is_torchdynamo_compiling() and self.device.type != input_ids.device.type:
	warnings.warn(
		"You are calling .generate() with the `input_ids` being on a device type different"
		f" than your model's device. `input_ids` is on {input_ids.device.type}, whereas the model"
		f" is on {self.device.type}. You may experience unexpected behaviors or slower generation."
		" Please make sure that you have put `input_ids` to the"
		f" correct device by calling for example input_ids = input_ids.to('{self.device.type}') before"
		" running `.generate()`.",
		UserWarning,
	)

# 8. prepare distribution pre_processing samplers
prepared_logits_processor = self._get_logits_processor(
	generation_config=generation_config,
	input_ids_seq_length=input_ids_length,
	encoder_input_ids=inputs_tensor,
	prefix_allowed_tokens_fn=prefix_allowed_tokens_fn,
	logits_processor=logits_processor,
	device=inputs_tensor.device,
	model_kwargs=model_kwargs,
	negative_prompt_ids=negative_prompt_ids,
	negative_prompt_attention_mask=negative_prompt_attention_mask,
)

# 9. prepare stopping criteria
prepared_stopping_criteria = self._get_stopping_criteria(
	generation_config=generation_config, stopping_criteria=stopping_criteria, tokenizer=tokenizer, **kwargs
)
```

Now, based on the `generation_mode` we extracted from generation config before, we can go into different generation mode. 

```python
# 10. go into different generation modes
if generation_mode == GenerationMode.ASSISTED_GENERATION:...
elif generation_mode == GenerationMode.DOLA_GENERATION:...
...
```

Basically it a bunch of  if...elif... codes that leads to different generation implementation. For llama, the generation mode is called `greedy_search`, so we'll just focus on that part of code. 

In this branch of generate. `prepared_logits_warper` is created, and input_ids is expanded. 

```python
# 11. prepare logits warper #NOTE: llama uses greedy search
prepared_logits_warper = (
	self._get_logits_warper(generation_config, device=input_ids.device)
	if generation_config.do_sample
	else None
)

# 12. expand input_ids with `num_return_sequences` additional sequences per batch
input_ids, model_kwargs = self._expand_inputs_for_generation(
	input_ids=input_ids,
	expand_size=generation_config.num_return_sequences,
	is_encoder_decoder=self.config.is_encoder_decoder,
	**model_kwargs,
)
```

Then a critical function named `_sample` is called, it will call `model.forward` for many times and complete the main part of sequence generation. 
```python
# 13. run sample (it degenerates to greedy search when `generation_config.do_sample=False`)
result = self._sample( #NOTE: model.forward wrapped in here
	input_ids,
	logits_processor=prepared_logits_processor,
	logits_warper=prepared_logits_warper,
	stopping_criteria=prepared_stopping_criteria,
	generation_config=generation_config,
	synced_gpus=synced_gpus,
	streamer=streamer,
	**model_kwargs,
)
```
Its parameters are **well documented** in source code. 

```python
"""
Generates sequences of token ids for models with a language modeling head using **multinomial sampling** and can be used for text-decoder, text-to-text, speech-to-text, and vision-to-text models.

Parameters:
	input_ids (`torch.LongTensor` of shape `(batch_size, sequence_length)`):
		The sequence used as a prompt for the generation.
		
	logits_processor (`LogitsProcessorList`):
		An instance of [`LogitsProcessorList`]. List of instances of class derived from [`LogitsProcessor`] used to modify the prediction scores of the language modeling head applied at each generation step.

	stopping_criteria (`StoppingCriteriaList`):
		An instance of [`StoppingCriteriaList`]. List of instances of class derived from [`StoppingCriteria`] used to tell if the generation loop should stop.

	generation_config ([`~generation.GenerationConfig`]):
		The generation configuration to be used as parametrization of the decoding method.

	synced_gpus (`bool`):
		Whether to continue running the while loop until max_length (needed for ZeRO stage 3)

	streamer (`BaseStreamer`, *optional*):
		Streamer object that will be used to stream the generated sequences. Generated tokens are passed through `streamer.put(token_ids)` and the streamer is responsible for any further processing.

	logits_warper (`LogitsProcessorList`, *optional*):
		An instance of [`LogitsProcessorList`]. List of instances of class derived from [`LogitsWarper`] used to warp the prediction score distribution of the language modeling head applied before multinomial sampling at each generation step. Only required with sampling strategies (i.e. `do_sample` is set in `generation_config`)

	model_kwargs:
		Additional model specific kwargs will be forwarded to the `forward` function of the model. If model is an encoder-decoder model the kwargs should include `encoder_outputs`.


Return:

	[`~generation.GenerateDecoderOnlyOutput`], [`~generation.GenerateEncoderDecoderOutput`] or `torch.LongTensor`:

	A `torch.LongTensor` containing the generated tokens (default behaviour) or a [`~generation.GenerateDecoderOnlyOutput`] if `model.config.is_encoder_decoder=False` and `return_dict_in_generate=True` or a [`~generation.GenerateEncoderDecoderOutput`] if `model.config.is_encoder_decoder=True`.
"""
```
So what does it do?

First, it initializes bunches of values that are essential for generation, including *setting up stopping_criteria, initialize attention, hidden_states and scores, doing extra work if we use encoder-dedcoder model*. 

Then a generation loop begins, its termination judged by a function called `self._has_unfinished_sequences`

```python
while self._has_unfinished_sequences(
	this_peer_finished, synced_gpus, device=input_ids.device, cur_len=cur_len, max_length=max_length

):
...
```

**Inside the generation loop**, model_inputs are first prepared, and then passed to model's `forward` function

```python
# prepare model inputs

model_inputs = self.prepare_inputs_for_generation(input_ids, **model_kwargs)

# prepare variable output controls (note: some models won't accept all output controls)
model_inputs.update({"output_attentions": output_attentions} if output_attentions else {})
model_inputs.update({"output_hidden_states": output_hidden_states} if output_hidden_states else {})

# forward pass to get next token
outputs = self(**model_inputs, return_dict=True)
```

We don't look into the detail of the forward function here. All we need to know is that `logits` can be extracted from `outputs`

```python
next_token_logits = outputs.logits[:, -1, :].clone()
```

Still remember the `prepared_logits_processor` and `prepared_logits_wrapper`? They are used here:

```python
# pre-process distribution
next_token_scores = logits_processor(input_ids, next_token_logits)
if do_sample:
	next_token_scores = logits_warper(input_ids, next_token_scores)
```

How does it work? I don't have time to see that yet, but we know that after this process, we get a score for each token(not just bare probabilities as logits is), then we can do the token selecton

```python
# token selection
if do_sample:
	probs = nn.functional.softmax(next_token_scores, dim=-1)
	next_tokens = torch.multinomial(probs, num_samples=1).squeeze(1)
else:
	next_tokens = torch.argmax(next_token_scores, dim=-1)
```
Token with the highest score would be appended to `input_ids`. This piece of code also tells what to do when generation is complete

```python
# finished sentences should have their next token be a padding token
if has_eos_stopping_criteria:
	next_tokens = next_tokens * unfinished_sequences + pad_token_id * (1 - unfinished_sequences)
# update generated ids, model inputs, and length for next step
input_ids = torch.cat([input_ids, next_tokens[:, None]], dim=-1)
if streamer is not None:
	streamer.put(next_tokens.cpu())
model_kwargs = self._update_model_kwargs_for_generation(
	outputs,
	model_kwargs,
	is_encoder_decoder=self.config.is_encoder_decoder,
)

unfinished_sequences = unfinished_sequences & ~stopping_criteria(input_ids, scores)
this_peer_finished = unfinished_sequences.max() == 0
cur_len += 1
```

After a lot of iterations(i.e. after complete response is generated), it returns a lot of parameters, but most importantly, returns `input_ids`


```python
if return_dict_in_generate:
	if self.config.is_encoder_decoder:
		return GenerateEncoderDecoderOutput(
			sequences=input_ids,
			scores=scores,
			logits=raw_logits,
			...
		)

	else:
		return GenerateDecoderOnlyOutput(
			sequences=input_ids,
			scores=scores,
			logits=raw_logits,
			...
		)

else:
	return input_ids
```

Then whatever it returns(`Dictionary` or `Tensor`) would just be returned to the caller of the generate function, so a call is completed.

>Actually, some tasks are still done before final return, like tackling a compatibility issue about the past_key_values, which is a `Tensor`(legacy) or an object of class `DynamicCache`(current). But it doesn't involve changing input_ids(at least for now)


Then we can convert this result to natural language using `tokenizer.decode`, which is very straightforward. 

# References
[github.com/huggingface/transformers](https://github.com/huggingface/transformers)

# After Writing

I'm really a rookie in Artificial Intelligence. Chewing transformers' source code is quite a challenge for me, but also a lot of fun. If anything above disagrees with what the code actually does, I would appreciate corrections from comment below. Further questions are also welcomed. 

