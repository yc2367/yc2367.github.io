---
layout: distill
title: "Quantization for LLM: A HW/SW Co-Design Perspective"
description: 
tags: Quantization CoDesign
giscus_comments: true
date: 2026-04-22
featured: true

authors:
  - name: Yuzong Chen
    affiliations:
      name: Cornell University

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).
toc:
  - name: I. Number Representation
    # if a section has subsections, you can add them as follows:
    subsections:
      - name: I-1. Integer Representation
      - name: I-2. Floating-Point Representation
  - name: II. Hardware of Number Representation
    subsections:
      - name: II-1. Integer Addition and Multiplication
      - name: II-2. Floating-Point Multiplication
      - name: II-3. Floating-Point Addition
  - name: III. Quantization Basics
    subsections:
      - name: III-1. Compute-Based vs. Memory-Based
      - name: III-2. Mathematical Background
      - name: III-3. Hardware Implication
  - name: IV. Advanced Quantization Techniques
    subsections:
      - name: IV-1. Block-Wise Quantization
      - name: IV-2. Scale Factor Quantization

# Below is an example of injecting additional post-specific styles.
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

In this post, I will discuss quantization, an essential technique for efficient LLM deployment on hardware. My goal is to give a tutorial about LLM quantization from an algorithm/hardware co-design perspective. I will start from binary number representations, quantization basics, to advanced quantization techniques, as well as their hardware implications<d-footnote>This blog is not meant to be a survey paper on LLM quantization. For a more comprehensive list of LLM quantization papers, you may want to check out <a href="https://github.com/Kai-Liu001/Awesome-Model-Quantization">this page</a>.</d-footnote>.



## I. Number Representation
Modern hardware is built to process binary digits, i.e., '0' and '1'. A number representation<d-footnote>In this blog, we only focus on the binary (base-2) number system. There are other number systems out there, such as octal and hexadecimal.</d-footnote>, also called number format, tells the hardware "How to interpret a binary number as a decimal number, and vice versa". In other words, a number representation defines a mapping between binary and decimal numbers. Modern hardware usually support two popular number representations: **Integer (INT)** and **Floating-Point (FP)**, each can have different precisions as illustrated below: 

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Number_Representation.png" width="80%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  


## <sub>I-1. Integer Representation</sub>

There are many integer number systems available out there. Some popular integer representations for deep learning are:
- **Sign-Magnitude Representation**, where an $$\mathrm{N}$$-bit binary number $$\mathrm{V}$$ has the following pattern and decimal value: 

$$\mathrm{V} \,=\, \mathrm{S\,{D_{N-2}\,D_{N-3}\,\dots\,D_{0}}} \,=\, (-1)^{\mathrm{S}} \,\cdot\, \left(\,\mathrm{D_{N-2}\,2^{N-2} \,+\, D_{N-3}\,2^{N-3} \,+\dots\,+\, D_{0}\,2^{0}}\,\right)$$

where $$\mathrm{S}$$ is the sign bit and $$\mathrm{D_{N-1}\,D_{N-2}\,\dots\,D_{0}}$$ represents the value's magnitude. The Sign-Magnitude Representation can be used to store quantized values under symmetric quantization. 

## <sub>I-2. Floating-Point Representation</sub>
Floating-point representation is the default number system used in deep learning training and inference, usually under 32-bit or 16-bit precision to ensure numerical stability and accuracy. A standard floating-point representation is characterized by three components: **Sign (S), Exponent (E$$), Mantissa (M$$)**. Since the sign always has a single bit, a floating-point representation with $$\mathrm{x}$$-bit exponent and $$\mathrm{y}$$-bit mantissa is often called $$\mathrm{\mathbf{ExMy}}$$, which has the following binary pattern and decimal value: 

$$\mathrm{V} \,=\, \mathrm{S \; \underbrace{E_{x-1}\dots E_{0}}_{\text{Exponent}} \; \underbrace{M_{y-1}\dots M_{0}}_{\text{Mantissa}}} \,=\, 
\begin{cases} 
\,(-1)^{\,\mathrm{S}} \,\cdot\, \mathrm{2^{E-B} \,\cdot\, 1.{M}} & \text{if } E\neq0 \\
\,(-1)^{\,\mathrm{S}} \,\cdot\, \mathrm{2^{1-B} \;\cdot\, 0.{M}} & \text{if } E=0 
\end{cases}$$

where $$\mathrm{E}$$ and $$\mathrm{M}$$ are interpreted as unsigned integers, the constant $$\mathrm{B = {2^{\,x-1}-1}}$$ is a fixed "bias" that allows the actual exponent to be both positive and negative. Note that there is a leading bit before $$\mathrm{M}$$, which makes the actual mantissa equal to $$1.{\mathrm{M}}$$ or $$0.\mathrm{M}$$. This leading bit is often called the hidden / implicit bit as it does not need to be stored. Instead, it can be determined during runtime by checking whether the value is normal $$(\mathrm{E} \neq 0)$$ or subnormal $$(\mathrm{E} = 0)$$.

In modern deep learning hardware, the default floating-point representations for lossless model training and inference are **FP32-E8M23** and **BF16-E8M7**<d-footnote>At 16-bit precision, BF16-E8M7 is preferred over the conventional IEEE FP16-E5M10 for training and deploying recent LLMs such as Llama-3 and Qwen-3, due to its wider numerical range that prevents overflow.</d-footnote>, where **BF16-E8M7** is usually adopted to store model weights and activations, and **FP32-E8M23** is used to store the intermediate results of various sensitive operators (e.g., partial sum of dot product and softmax). 



## II. Hardware of Number Representation
Given the importance of quantization on modern deep learning hardware, I think it is necessary for me to explain "How a Multiply-Accumulate (MAC), the key operation of deep learning, is carried out on hardware under different number representations." Because a MAC consists of multiplication and addition, I will talk about the hardware implication of these two primitive operations under integer and floating-point representations. 

## <sub>II-1. Integer Addition and Multiplication</sub>

The simplest [binary integer adder](https://en.wikipedia.org/wiki/Adder_(electronic) on hardware uses Ripple-Carry Adder, with both area and time complexity equal to $$\mathrm{O(N)}$$<d-footnote>This bound is derived by assuming that a binary full-adder has area and time complexity of O(1).</d-footnote>, where $$\mathrm{N}$$ is the integer precision. Other variants of binary adders include Carry-Select Adder, Carry-Lookahead Adder, etc., with different area and delay trade-offs.

The [binary integer multiplier](https://en.wikipedia.org/wiki/Binary_multiplier) is usually made by cascading many integer adders with an area complexity of $$\mathrm{O(N^2)}$$. A typical integer multiplier multiplies every bits of two multiplicands via simple logical operations, followed by reducing these binary partial products through adder chains. Modern integer multipliers adopt efficient implementations such as Baugh–Wooley algorithm, Wallace Tree, or Dadda Tree for fast reduction of partial products in a single clock. For example, an efficient multiplier like Baugh–Wooley only has a time complexity of $$\mathrm{O(2N)}$$.

## <sub>II-2. Floating-Point Multiplication</sub>
The floating-point MAC is rather complicated as it needs to deal with exponent. To simply the discussion, I will only focus on *normal* floating-point numbers, whose binary exponent field is larger than zero. 

Consider two normal floating-point numbers $$\mathrm{F}_{1}$$ and $$\mathrm{F}_{2}\,$$: 

$$\begin{aligned}
\mathrm{F_1} \,&=\, (-1)^{\,\mathrm{S}_{1}} \,\cdot\, 2^{\mathrm{E}_{1}-\mathrm{B}} \,\cdot\, 1.\mathrm{M}_{1} \\
\mathrm{F_2} \,&=\, (-1)^{\,\mathrm{S}_{2}} \,\cdot\, 2^{\mathrm{E}_{2}-\mathrm{B}} \,\cdot\, 1.\mathrm{M}_{2}
\end{aligned}
$$

The multiplication product is: 

$$\mathrm{P} = \mathrm{F_1} \cdot \mathrm{F_2} \,=\, (-1)^{\mathrm{S}_{1\,} \oplus\, \mathrm{S}_{2}} \,\cdot\, 2^{\mathrm{E}_{1} + \mathrm{E}_{2}-\mathrm{2B}} \,\cdot\, \underbrace{(\,1.\mathrm{M}_{1} \times 1.\mathrm{M}_{2}\,)}_{\text{Un-norm Mantissa}}
$$

Based the definition of floating-point representation, as described previously, the sign, exponent, and mantissa of a product should be: 

$$\begin{aligned}
\mathrm{S_{P}} \,&=\, \mathrm{S}_{1} \oplus\, \mathrm{S}_{2} \\[0.3em]
\mathrm{M_{P}} \,&=\, \begin{cases} 
\; (\,1.\mathrm{M}_{1} \times 1.\mathrm{M}_{2}\,) & \text{if un-norm mantissa} \leq 2 \\[0.1em]
\;(\,1.\mathrm{M}_{1} \times 1.\mathrm{M}_{2}\,) \;/\; 2 & \text{if un-norm mantissa} > 2
\end{cases} \\[0.3em]
\mathrm{E_{P}} \,&=\, \begin{cases} 
\;\mathrm{E}_{1} + \mathrm{E}_{2}-\mathrm{B} & \quad \text{if un-norm mantissa} \leq 2 \\[0.1em]
\;\mathrm{E}_{1} + \mathrm{E}_{2}-\mathrm{B} + 1 & \quad \text{if un-norm mantissa} > 2 \\
\end{cases} \\
\end{aligned}
$$

The above formula shows that:
- The product sign bit is simply a logical XOR between the sign bits of $$\mathrm{F}_{1}$$ and $$\mathrm{F}_{2}\,$$.
- The product mantissa can be obtained via unsigned integer multiplication between the mantissa $$1.\mathrm{M}_{1}$$ and $$1.\mathrm{M}_{2}\,$$. A conditional check (a.k.a normalization) is needed to determine whether the un-normalized mantissa product is larger than 2. If yes, the un-normalized mantissa should be divided by 2, i.e., normalized to the range $$[1, 2)$$.
- The product exponent can be obtained via unsigned integer addition / subtraction between the exponents $$\mathrm{E}_{1}\,$$and $$\mathrm{E}_{2}\,$$, and the bias $$\mathrm{B}\,$$<d-footnote>Note that we only subtract a single bias B from (E<sub>1</sub> + E<sub>2</sub>), since the standard floating-point representation will automatically subtract another exponent bias.</d-footnote>. 
As discussed before, the integer adder and multiplier have area complexity of $$\mathrm{O}(N)$$ and  $$\mathrm{O}(N^2)$$, respectively. Given that the cost of XORing the sign bit is negligible, the area complexity of a $$\mathrm{\mathbf{ExMy}}$$ floating-point representation is roughly $$\mathrm{O(\,2x \,+\,(y+1)^2\,)}$$. The following table shows the synthesized combinational logic area of BF16, FP16, and FP32 multipliers without the normalization block, under 28nm technology. It can be observed that the area complexity serves as a good estimation of the relative multiplier area between different floating-point representations.
<table style="width: 60%; border-spacing: 5px; margin-left: 1.5em; margin-top: -1.2em; margin-bottom: 1.5em;"><thead>
  <tr>
    <th colspan="1"> <br>Representation </th>
    <th colspan="1"> <br>Area Complexity </th>
    <th colspan="1"> <br>Multiplier Area </th>
  </tr></thead>
<tbody>
  <tr>
    <td> BF16-E8M7 </td>
    <td> 80    (1×) </td>
    <td> 167.2 um<sup>2</sup>    (1×) </td>
  </tr>
  <tr>
    <td> FP16-E5M10 </td>
    <td> 131   (1.64×) </td>
    <td> 260.9 um<sup>2 </sup>  (1.56×) </td>
  </tr>
  <tr>
    <td> FP32-E8M23 </td>
    <td> 592  (7.4×) </td>
    <td> 1166.8 um<sup>2</sup>  (6.98×) </td>
  </tr>
</tbody></table>


## <sub>II-3. Floating-Point Addition</sub>
Again, consider two normal floating-point numbers $$\mathrm{F}_{1}$$ and $$\mathrm{F}_{2}\,$$: 

$$\begin{aligned}
\mathrm{F_1} \,&=\, (-1)^{\,\mathrm{S}_{1}} \,\cdot\, 2^{\,\mathrm{E}_{1}-\mathrm{B}} \,\cdot\, 1.\mathrm{M}_{1} \\
\mathrm{F_2} \,&=\, (-1)^{\,\mathrm{S}_{2}} \,\cdot\, 2^{\,\mathrm{E}_{2}-\mathrm{B}} \,\cdot\, 1.\mathrm{M}_{2}
\end{aligned}
$$

Let's assume $$\mathrm{E}_{1} \geq \mathrm{E}_{2\,}$$, then $$\mathrm{F}_{2}$$ can be re-written as: 

$$\mathrm{F_2} \,=\, (-1)^{\,\mathrm{S}_{2}} \,\cdot\, 2^{\,\mathrm{E}_{1}-\mathrm{B}} \,\cdot\, \left(\,1.\mathrm{M}_{2} \;/\; 2^{\,\mathrm{E}_{1}-\mathrm{E}_{2}}\,\right) \,=\, (-1)^{\,\mathrm{S}_{2}} \,\cdot\, 2^{\,\mathrm{E}_{1}-\mathrm{B}} \,\cdot\, \left(\,1.\mathrm{M}_{2} \gg (\mathrm{E}_{1}-\mathrm{E}_{2})\,\right)$$

The above step is called "Exponent Alignment", where two numbers' exponents are compared and aligned to the larger exponent. Since we increase the exponent of $$\mathrm{F}_{2}$$ by $$(\,\mathrm{E}_{1}-\mathrm{E}_{2}\,)$$, its mantissa needs to be divided by $$2^{\,\mathrm{E}_{1}-\mathrm{E}_{2}}$$, which is equivalent to right-shifting the mantissa by $$(\,\mathrm{E}_{1}-\mathrm{E}_{2}\,)$$ bits. 

The addition sum can then be expressed as: 

$$\mathrm{A} = \mathrm{F_1} + \mathrm{F_2} \,=\, 2^{\,\mathrm{E}_{1} - \mathrm{B}} \,\cdot\, \underbrace{\left[\; (-1)^{\,\mathrm{S}_{1}} \cdot 1.\mathrm{M}_{1} \;+\;  (-1)^{\,\mathrm{S}_{2}} \cdot \left(\,1.\mathrm{M}_{2} \gg (\mathrm{E}_{1}-\mathrm{E}_{2})\,\right) \;\right]}_{\text{Un-norm Mantissa}}
$$

The sum's sign and mantissa are derived from the above un-normalizaed mantissa through a hardware normalization block, which does a lot of condition checking to ensure that the final mantissa is normalized to the range $$[1, 2)$$.

Based on the above analysis, you may notice that it's not easy to derive a good upper bound for the adder's area complexity. There are also many low-level design considerations that vary across different hardware vendors when they design a floating-point adder. For example, how many bits should we reserve for accumulating the right-shifted mantissa after exponent alignment? 

$$1.\mathrm{M}_{2} \gg (\mathrm{E}_{1}-\mathrm{E}_{2})$$

At FP32-E8M23, the largest and smallest representable exponents are $$127$$ and $$-126$$, respectively. This means that the 24-bit mantissa (including the hidden bit) can be right-shifted by up to $$253$$ bits, leading to a total precision of $$277$$ bits for accumulating two aligned mantissa, which is clearly not a practical choice. Thus, in reality, hardware vendors typically store the shifted mantissa using much less precision, which can often be found through reverse engineering. This limited precision creates a trade-off between numerical accuracy and hardware cost: allocating less precision to accumulate the shifted mantissa reduces hardware cost but harms numerical accuracy<d-footnote>For instance, <a href="https://arxiv.org/abs/2412.19437">DeepSeek-V3</a> finds that on NVIDIA H800 GPUs, the accumulation precision of FP8 MAC is only 14 bits, which can introduce a maximum relative error of nearly 2% compared to using FP32 accumulation precision.</d-footnote>.



## III. Quantization Basics
Let's now jump into quantization: one of the most popular methods to improve the hardware efficiency of LLMs. 

## <sub>III-1. Compute-Based vs. Memory-Based</sub>
Before diving into technical details, I would like to classify existing quantization methods into two broad categories: **compute-based** and **memory-based**, depending on how they interpret the low-precision operands in hardware<d-footnote>This is only one of many ways to classify quantization methods. For instance, one could also classify different methods based on their target quantized operands, e.g., weight-only quantization and weight-activation quantization.</d-footnote>. Below is a visualization:
<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Memory_vs_Compute_Quantization.png" width="60%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  
- Compute-based Quantization: The retrieved low-bit data from memory **can** be directly used to perform MAC. Examples include quantizing a tensor from BF16 to INT8 / FP8 / FP4, etc.
- Memory-based Quantization: The retrieved low-bit data from memory **cannot** be used to perform MAC. Instead, the low-bit data serves as an index to access another memory, which is often called a codebook and stores high-precision data (e.g., in BF16). The codebook access returns the actual data that is used to perform MAC.

Compute-based quantization enables computation on low-bit MAC hardware, whereas memory-based quantization requires an additional memory access to the codebook cache before performing computation on BF16 MAC hardware. Hence, at the same model compression ratio, compute-based quantization typically consumes significantly less area and energy, making it the favorable approach in modern LLMs<d-footnote>Examples include DeepSeek-V3, DeepSeek-R1, Qwen3.5, GPT-OSS, etc.</d-footnote> and AI chips<d-footnote>Examples include NVIDIA GPU, Meta MTIA, Tesla AI4, Microsoft MAIA, etc.</d-footnote>. **In the remaining of this blog, I will focus on discussing compute-based quantization.** 

## <sub>III-2. Mathematical Background of Quantization</sub>
Given a tensor represented in high-precision floating-point format (e.g., BF16), the purpose of quantization is to convert this tensor to a low-precision number format (e.g., INT8 / FP8), where every tensor element is mapped to its closest quantization value. However, directly mapping a value to another representation can introduce many issues, such as **overflow** and **underflow**, which are illustrated below: 

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Overflow_Underflow.png" width="90%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div> 

The above example quantizes a tensor of four elements to the 3-bit sign-magnitude integer (INT3) representation, and measures the resulting root-mean-square (RMS) quantization error. As shown in the top-left figure, if the original tensor contains very small values that are close to zero, directly casting to INT3 will make all values underflow to zero, resulting in catastrophic information loss. Similarly, as shown in the bottom left-figure, if the original tensor contains very large values that overflow the INT3 range, directly casting to INT3 will clamp these large values to the endpoint of INT3, resulting in significant saturation error. 

To address the overflow / underflow issues, standard quantization linearly transforms / **"scales"** the original tensor to the quantized value range, then rounds each scaled tensor value to its nearest quantization value. After quantization, a re-scaling / dequantization step is performed at the end to map the quantized tensor back to the original tensor's range. This procedure is summarized by the following equations: 

$$\mathrm{S} = \frac{ |\mathrm{X}|_{\text{max}} }{ \mathrm{Q}_{\text{max}} }  ; \ \ \
\mathrm{X_\mathrm{S}} = \frac{\mathrm{X}}{\mathrm{S}} ; \ \ \
\mathrm{X_Q} = \texttt{Round}\left(\,\mathrm{X_\mathrm{S}}\,,\, \mathrm{Q}\,\right) ; \ \ \
\mathrm{X_\mathrm{D}} = \mathrm{X_Q} \cdot \mathrm{S}
$$

where $$\mathrm{X}$$ is the original high-precision tensor, $$\mathrm{Q}$$ is the set of quantization values defined by a low-bit number representation, $$\mathrm{S}$$ is a high-precision scale factor, $$\mathrm{X_{S}}$$ is the scaled tensor, $$\mathrm{X_{Q}}$$ is the quantized tensor, and $$\mathrm{X_{D}}$$ is the dequantized tensor. As shown in the above example, scaling can effectively mitigates the impact of overflow / underflow. When the tensor values are very small, scaling can enlarge the tensor range so that many elements are mapped to meaningful quantization values. When the tensor values are very large, scaling can prevent large saturation error. 

Because of scaling, if two sets of quantization values, $$\mathrm{Q_{1}}$$ and $$\mathrm{Q_{2}}$$, differ only by a constant factor, then they produce mathematically equivalent results after dequantization: 

$$\begin{aligned}
\text{Assume}: \quad &\mathrm{Q_{2}} = c\,\mathrm{Q_{1}} \\[0.3em]
\text{With }\mathrm{Q_{1}}: \quad &
\mathrm{S_{1}} = \frac{ |\mathrm{X}|_{\text{max}} }{ \mathrm{Q}_{1,\text{max}} }  ; \ \
\mathrm{X_{S_{1}}} = \frac{\mathrm{X}}{\mathrm{S_{1}}} ; \ \
\mathrm{X_{Q_{1}}} = \texttt{Round}\left(\mathrm{X_{S_{1}}}, \mathrm{Q_{1}}\right) ; \ \
\mathrm{X_{D_{1}}} = \mathrm{X_{Q_{1}}} \cdot \mathrm{S_{1}} \\[0.3em]
\text{With }\mathrm{Q_{2}}: \quad &
\mathrm{S_{2}} = \frac{ \mathrm{S_{1}} }{ c }  ; \ \
\mathrm{X_{S_{2}}} = \frac{c\,\mathrm{X}}{\mathrm{S_{1}}} ; \ \
\mathrm{X_{Q_{2}}} = \texttt{Round}\left(c\,\mathrm{X_{S_{1}}}, \mathrm{c\,Q_{1}}\right) = c\,\mathrm{X_{Q_{1}}} ; \ \
\mathrm{X_{D_{2}}} = \mathrm{X_{Q_{2}}} \cdot \mathrm{S_{2}} = \mathrm{X_{D_{1}}} \\[0.3em]
\end{aligned}
$$

For example, at 4-bit quantization, you might have seen some papers using a format called E1M2, which has the set of quantization values $$\pm\{\,0,\, 0.25,\, 0.5,\, 0.75,\, 1,\, 1.25,\, 1.5,\, 1.75\,\}$$. This is actually equivalent to INT4 quantization with the set of values $$\pm\{\,0,\, 1,\, 2,\, 3,\, 4,\, 5,\, 6,\, 7\,\}$$.

## <sub>III-3. Hardware Implication of Quantization</sub>
By reducing the operand bit-width, quantization not only decreases the memory footprint, but also enables more efficient computation using low-precision MAC hardware. When the baseline LLM stores model weights and input activations in BF16, its linear layer will perform a GEMM using BFP16 multiply and FP32 accumulation. As an example, if we employ per-tensor quantization to convert model weights and input activations to INT8, the linear layer can instead perform a GEMM using INT8 multiply and INT32 accumulation, followed by dequantization that multiplies the output matrix with two scale factors. This is visualized below: 

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Quantized_GEMM.png" width="90%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div> 

To understand the hardware implication of these two approaches, we can break down the hardware cost into two parts: memory footprint and computational efficiency. The memory can be quantified as the total number of bytes required to store weights, inputs, and outputs. Accordingly, the memory costs of baseline and quantized GEMMs are: 

$$\begin{aligned}
\text{Baseline:} & \ \ 2MK + 2KN + 4MN \\
\text{Quantized:} & \ \ MK + KN + 4MN
\end{aligned}$$

For computation, recall that a GEMM with size $$M\times K \times N$$ requires $$MKN$$ MAC operations. For dequantization, the scale factors $$\mathrm{S_{W}}$$ and $$\mathrm{S_{A}}$$ can be pre-multiplied and stored as a constant. Multiplying this constant with the output matrix requires $$MN$$ multiplications. Thus, the computation costs of baseline and quantized GEMMs are: 

$$\begin{aligned}
\text{Baseline:} & \ \ MKN \cdot \text{MAC}_{\text{BF16-FP32}} \\
\text{Quantized:} & \ \ MKN \cdot \text{MAC}_{\text{INT8-INT32}} \,+\, (MN+1) \cdot \text{MUL}_{\text{FP32}} 
\end{aligned}$$

In modern LLMs, the $$K$$-dimension is usually very large (e.g., several thousands), making the cost of dequantization negligible compared to MAC. Consequently, the compute cost is roughly proportional to the cost of MAC units. The following table summarizes the synthesized area and energy of MAC units at different input-output precisions under 28nm technology. It can be observed that integer quantization can significantly reduce the computation cost compared to high-precision floating-point operations<d-footnote>In some cases, a quantization strategy chooses to maintain the original 16-bit precision to avoid unacceptable accuracy loss, and instead uses a more hardware-efficient number representation to reduce the computation cost. For instance, INT16 does not save memory cost over BF16, but it can reduce the area and energy of MAC by ~1.8×.</d-footnote>. 

<table style="width: 70%; border-spacing: 5px; margin-left: 1.5em; margin-top: -1em; margin-bottom: 1.5em;"><thead>
  <tr>
    <th rowspan="2"> <br>Multiplier </th>
    <th rowspan="2"> <br>Accumulator </th>
    <th colspan="2"> <br>Area (um<sup>2</sup>) </th>
    <th colspan="2"> <br>Energy (pJ) </th>
  </tr>
  <tr>
    <th colspan="1"> <br>Value </th>
    <th colspan="1"> <br>Ratio </th>
    <th colspan="1"> <br>Value </th>
    <th colspan="1"> <br>Ratio </th>
  </tr></thead>
<tbody>
  <tr>
    <td> INT8 </td><td> INT32 </td><td>213.8 </td><td>1× </td><td>0.16 </td><td>1× </td>
  </tr>
  <tr>
    <td> INT8 </td><td> INT48 </td><td>286.2  </td><td>1.34× </td><td>0.22 </td><td>1.38× </td>
  </tr>
  <tr>
    <td> INT16 </td><td> INT48 </td><td>516.2 </td><td>2.41× </td><td>0.36 </td><td>2.25× </td>
  </tr>
  <tr>
    <td> BF16 </td><td> FP32 </td><td>929.4 </td><td>4.35× </td><td>0.63 </td><td>3.94× </td>
  </tr>
  <tr>
    <td> FP32 </td><td> FP32 </td><td>1994.7 </td><td>9.33× </td><td>1.28 </td><td>8× </td>
  </tr>
</tbody></table>



## IV. Advanced Quantization Techniques
In this section, I will discuss several techniques for improving the performance of quantized LLM inference. Many of these techniques are widely adopted in SoTA LLMs and AI chips. **From an algorithm-hardware co-design perspective, a good quantization strategy introduces a trade-off between model accuracy, memory footprint, and computational efficiency.** These trade-offs will also be examined throughout this section. 

## <sub>IV-1. Block-Wise Quantization</sub>
Block-wise quantization is employed in many recent LLMs such DeepSeek-V3, GPT-OSS, and Kimi-K2. Before diving into this technique, let's first recap the concept of outliers. An outlier is defined as a value whose magnitude is significantly larger than the rest of values in a tensor. It is well known that LLM tensors contain extreme outliers, and preserving the numerical fidelity of these outliers is the key to reducing quantization error and improving model performance<d-footnote>For more details, interested readers can checkout <a href="https://arxiv.org/abs/2208.07339">this paper</a>.</d-footnote>. 

Quantization granularity measures how many elements are scaled and quantized together. For example, the coarsest quantization granularity is simply **Tensor-Wise Quantization**. But at this granularity, even a single outlier can significantly squeeze most of the remaining values, making them underflow to zero after quantization. This phenomenon is visualized in the following top-left figure, which uses INT4 sign-magnitude representation to quantize a $$3\times4$$ tensor. To mitigate the impact of outliers, we can reduce the quantization granularity so that fewer values are quantized together. For example, **Row-Wise Quantization** applies independent scaling to each row of a tensor, limiting the effect of an outlier to values within the same row, as illustrated in the following top-right figure. On top of row-wise quantization, the granularity can be further reduced by partitioning each row into smaller segments / blocks, an approach called **Block-Wise Quantization**, which further constrains the impact of outliers to a small block. This is illustrated in the following bottom figure, where a row is partitioned into two blocks:

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Quantization_Granularity.png" width="100%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  

**Why Does Block-Wise Quantization Help?** 
Let's analyze how quantization affects the maximum element $$\mathrm{X}_{\text{max}}$$<d-footnote>Here, the "maximum element" refers to the element with maximum magnitude, thus can be either positive or negative.</d-footnote>, without caring about granularity. In addition, let's assume the quantized number representation is symmetric with respect to zero (i.e., $$\mathrm{Q}_{\text{max}} = -\mathrm{Q}_{\text{min}}\,$$), which holds for most number representations such as floating-point and sign-magnitude integer. 
According to the definition of quantization, we have: 

$$\begin{aligned}
\mathrm{S} &= \frac{ |\mathrm{X}|_{\text{max}} }{ \mathrm{Q}_{\text{max}} } = \frac{ |\mathrm{X}_{\text{max}}| }{ \mathrm{Q}_{\text{max}} } \\[0.3em]
\mathrm{X_{\text{max},\,S}} &= \frac{ \mathrm{X}_{\text{max}}}{\mathrm{S}} = \frac{ \mathrm{X}_{\text{max}}\cdot \mathrm{Q}_{\text{max}} }{ |\mathrm{X}_{\text{max}}| } = \mathrm{X}_\text{sign} \cdot \mathrm{Q}_{\text{max}}  \\[0.3em]
\mathrm{X_{\text{max},\,Q}} &= \texttt{Round-To-Nearest}\left(\,\mathrm{X_{\text{max},\,S}}\,,\, \mathrm{Q}\,\right) = \mathrm{X}_\text{sign} \cdot \mathrm{Q}_{\text{max}}  \\[0.3em]  
\mathrm{X_{\text{max},\,D}} &= \mathrm{X_{\text{max},\,Q}} \cdot \mathrm{S} = \left(\,\mathrm{X}_\text{sign} \cdot \mathrm{Q}_{\text{max}}\,\right) \,\cdot\, \frac{ |\mathrm{X}_{\text{max}}| }{ \mathrm{Q}_{\text{max}} } = \mathrm{X}_{\text{max}}
\end{aligned}$$

The above last equation unveils a critical insight: **If the scale factor is accurately computed (e.g., using high precision such as FP32), then the original maximum element remains unchanged after dequantization, regardless of the quantization granularity.** In other words, the maximum element of a granularity has no quantization error. Therefore, under a finer quantization granularity, more maximum elements can be represented accurately, leading to lower total quantization error. 

**Algorithm-Hardware Trade-Off under Different Granularity:**
As discussed above, an obvious benefit of reduced quantization granularity is lower quantization error, which potentially offers better model accuracy. However, this algorithmic benefit comes with additional computation and memory costs. Below, I try to visualize the computation flow of performing a $$M\times K \times N$$ GEMM under different INT4 quantization granularity:
<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/Quantization_Granularity_Hardware.png" width="100%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  
- **Tensor-Wise Quantization**: This is the coarsest granularity, where the GEMM requires $$MKN$$ low-precision MAC operations. For dequantization, the FP32 scale factors $$\mathrm{S_{W}}$$ and $$\mathrm{S_{A}}$$ can be pre-multiplied and stored as a constant, and multiplying this constant with the output matrix requires $$MN$$ FP32 multiplications.
- **Row-Wise Quantization**: The whole GEMM still requires $$MKN$$ low-precision MAC operations. But for dequantization, the row-wise scale factors of weight and activation are vectors of size of $$M \times 1$$ and $$N \times 1$$, respectively. These two vectors first perform an outer product, which is then element-wise multiplied with the output matrix, leading to $$2MN$$ FP32 multiplications.
- **Block-Wise Quantization**: Since a matrix row is partitioned into multiple blocks, the complete row-wise dot product will contain $$B$$ partial sums, where $$B$$ is the number of blocks. This means that the whole GEMM is partitioned into $$B\times$$ small GEMMs, each of size $$(M \times K/B \times N)$$ with row-wise quantization (a block of the original large GEMM can be viewed as a row of the small GEMM). The total number of low-precision MAC operations remains $$B \times (M \times K/B \times N) = MKN$$. The tricky part comes to the dequantization process: every small GEMM requires $$2MN$$ multiplications as in row-wise quantization, leading to a total of $$2BMN$$ FP32 multiplications. But the dequantized output matrix of every small GEMM only forms a partial sum, which also needs to be reduced over all blocks, leading to a total of $$(B-1)MN$$ extra FP32 additions.

One thing I didn't explain in the above figure, which you might find a little weird, is the output matrix precision. When I talk about BF16 GEMM and INT8 GEMM in previous sections, I used FP32 and INT32 as the output precision. However, the output matrix precision can be reduced according to the input precision, which in turn reduces the hardware accumulator precision and cost. Taking the above example, the sign-magnitude INT4 representation has a numerical range of $$[-7, 7]$$, and multiplying two INT4 numbers yields a range of $$[-49, 49]$$. Consider a trillion-parameter LLM such as [Kimi-K2.5](https://huggingface.co/moonshotai/Kimi-K2.5/blob/main/config.json), whose maximum dot product size of $$18432$$. Under tensor-wise and row-wise INT4 quantization<d-footnote>Here, I am only using Kimi-K2.5 as an illustrative example to describe how we can optimize the hardware accumulator precision considering the quantization precision and dot product size. In practice, Kimi-K2.5 keeps input activations in BF16 and applies INT4 block-wise quantization to weights. More generally, tensor-wise or row-wise quantization is rarely used at very low precisions such as 4 bits.</d-footnote>, the INT4 dot product output has a range of $$[-903168, 903168]$$, which can be represented with a 21-bit sign-magnitude integer<d-footnote">In practice, the output accumulator precision may be further optimized depending on the data statistics, as it's impossible that all quantized INT4 values are -7 and 7.</d-footnote>. Similarly, for block-wise INT4 quantization under a typical block size of 128, the dot product output has a range of $$[-6272, 6272]$$, which can be represented with a 14-bit sign-magnitude integer.

The memory costs of different quantization granularity are much simpler to analyze. In addition to weight, input, and output, quantization stores extra metadata such as the scale factors. For tensor-wise and row-wise quantization, the memory cost of scale factors is pretty negligible when the dot product size is large enough, which is the case for modern LLMs. However, this assumption may not hold for block-wise quantization. For example, with a block size of $$128$$, an FP32 scale factor introduces an overhead of $$32/128 = 0.25$$ bits per element. As the block size further decreases (to improve model performance), the storage overhead of scale factors can no longer be ignored. Thus, for the next quantization technique, I will discuss how to reduce the overhead of scale factors. 

## <sub>IV-2. Scale Factor Quantization</sub>
While block-wise quantization brings algorithmic benefits by reducing quantization error, it incurs additional hardware cost from storing numerous FP32 scale factors and performing more FP32 multiplications / additions during dequantization. 

The question is: **How can we reduce the cost of scale factors? The answer is surprisingly  straightforward: if quantization can reduce the memory and computation costs of tensor elements, why not apply it to scale factors?** Building on this idea, the industry has proposed two methods to quantize the block scale factor: [Microscaling (MX)](https://www.opencompute.org/documents/ocp-microscaling-formats-mx-v1-0-spec-final-pdf) and [NVIDIA’s (NV)](https://developer.nvidia.com/blog/introducing-nvfp4-for-efficient-and-accurate-low-precision-inference/) approach. Both quantize the scale factor to 8 bits, but using different number representations<d-footnote>Besides the scale factor representation, MX and NV also set fixed block sizes of 32 and 16, respectively. However, the concept of block size is not strictly tied to these two approaches. For instance, one can choose to reduce the MX block size from 32 to 16, as did in <a href="https://arxiv.org/abs/2603.08713">this paper from Meta</a>. Therefore, I will focus on discussing the scale factor representation without caring too much about the block size.</d-footnote>: MX uses the 8-bit power-of-two format (E8M0), whereas NV uses the 8-bit floating-point format with 4-bit exponent and 3-bit mantissa (FP8-E4M3). Below is a visualization of how MX and NV represent scale factors, assuming the block elements are quantized to FP4 (i.e., the popular MXFP4 and NVFP4 formats): 

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/MXFP4_NVFP4_Overview.png" width="75%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div> 

<h3 style="margin-top: 1rem; margin-bottom: -0.25rem;">Microscaling (MX) Approach</h3>
MX is standardized by the Open Compute Project, and widely supported by recent AI chips such as AMD Radeon, Meta MTIA, and Microsoft MAIA. In this approach, the high-precision scale factor is quantized to power-of-two, as described in the following equations: 

$$
\mathrm{S} = \frac{ |\mathrm{B}|_{\text{max}} }{ \mathrm{Q}_{\text{max}} } ; \ \ 
\boldsymbol{\mathrm{S_{Q}}} = 2^{\left\lceil\,\texttt{log2}\left( \,\mathrm{S} \,\right) \,\right\rceil} ; \ \ 
\mathrm{B_\mathrm{S}} = \frac{\mathrm{B}}{\boldsymbol{\mathrm{S_{Q}}}} ; \ \
\mathrm{B_Q} = \texttt{Round}\left(\,\mathrm{B_\mathrm{S}}\,,\, \mathrm{Q}\,\right) ; \ \
\mathrm{B_\mathrm{D}} = \mathrm{B_Q} \cdot \boldsymbol{\mathrm{S_{Q}}}
$$

where $$\mathrm{B}$$ is the block to be quantized. After calculating the scale factor $$\mathrm{S}$$, we take its binary logarithm followed by a ceiling operation to obtain the integer exponent. The quantized scale factor is the base-2 exponentiation of this integer exponent. The ceiling operation ensures that $$\mathrm{S}$$ is always *rounded up* to the nearest power-of-two that is larger than or equal to $$\mathrm{S}$$. This is different from the rounding mechanism for the scaled block element $$\mathrm{B_{S}\,}$$, which is *rounded (up or down)* to the nearest quantization value.

The purpose of rounding up during power-of-two scale factor quantization, is to ensure the scaled block $$\mathrm{B_{S}}$$ can always remains within the representable quantization range without overflow. Specifically, we want to ensure that: 

$$\begin{aligned}
|\mathrm{B_{S}}| \leq \mathrm{Q}_{\text{max}} 
\iff |\mathrm{B_{\text{max},\,S}}| \leq \mathrm{Q}_{\text{max}} 
\iff \frac{|\mathrm{B_{\text{max}}}|}{\mathrm{S_{Q}}} \leq \mathrm{Q}_{\text{max}} 
\iff \mathrm{S_{Q}} \geq \frac{|\mathrm{B_{\text{max}}}|}{\mathrm{Q}_{\text{max}}}
\iff \mathrm{S_{Q}} \geq\, \mathrm{S}
\end{aligned}$$

Thus, to avoid overflow, the quantized scale factor should be larger than or equal to the original scale factor. 

Interested readers may ask: **Why don't we use the normal rounding (up or down) mechanism when rounding the scale factor to its nearest power-of-two?** In fact, this is exactly what [the original MX quantizer from Microsoft](https://github.com/microsoft/microxcaling/blob/7bc41952de394f5cc5e782baf132e7c7542eb4e4/mx/mx_ops.py#L173) does. However, this naive approach can cause significant overflow of the scaled block, $$\mathrm{B_{S}\,}$$, leading to large saturation error<d-footnote>Hence, you should never refer to that original implementation for MX quantization.</d-footnote>. To better explain this problem, I will use the popular MXFP4 format as an example, where each block is quantized to FP4-E2M1 that can represent $$\{\,0\,, \pm\,0.5\,, \pm\,1\,, \pm\,1.5\,, \pm\,2\,, \pm\,3\,, \pm\,4\,, \pm\,6\,\}$$. Let's analyze what happens to the block maximum, $$\mathrm{B}_{\text{max\,}}$$, when the scale factor is rounded (up or down) to its nearest power-of-two. For simplicity, I will assume $$\mathrm{B}_{\text{max}}$$ is a positive normal FP32 value, leading to the following equations to compute $$\mathrm{B_{S}\,}$$: 

$$
\mathrm{S} = \frac{\mathrm{B}_{\text{max}}}{6} ; \ \  
\mathrm{S_{Q}} = 2^{\left\lfloor\,\texttt{log2}\left( \,\mathrm{S} \,\right) \,\right\rceil} ; \ \ 
\mathrm{B_{\text{max},\,S}} = \frac{\mathrm{B}_{\text{max}}}{\mathrm{S_{Q}}}$$

Since $$\mathrm{B}_{\text{max}}$$ is a positive normal FP32 value, I can write it in floating-point representation as described in Section-I.2: 

$$
\mathrm{B_{\text{max}}} = 0\;\mathrm{\underbrace{E_{7}\dots E_{0}}_{\text{Exponent}} \; \underbrace{M_{22}\dots M_{0}}_{\text{Mantissa}}} = (-1)^0 \cdot 2^{\mathrm{E}-127} \cdot (\,1+\mathrm{M}\,) = 2^{\mathrm{E\,'}} \cdot \mathrm{M\,'}
$$

where $$\mathrm{E\,'}$$ is the actual exponent after subtracting the bias $$127$$, and $$\mathrm{M\,'}$$ is the actual mantissa after adding the hidden bit $$1$$ for normal values. Using this representation, the scale factor can be expressed as: 

$$
\mathrm{S} = \frac{ 2^{\,\mathrm{E\,'}} \cdot \mathrm{M\,'} }{ 2^2 \cdot 1.5 } =  2^{\,\mathrm{E\,'} - \,2} \cdot \frac{\mathrm{M\,'}}{1.5}
$$

Now, assume $$\mathrm{M\,'} > 1.5$$, which accounts for 50% of possible cases<d-footnote>Assume the mantissa is uniformly distributed between [1.0, 2.0), then we have 50% probability that the mantissa is larger than 1.5</d-footnote>, and since $$\mathrm{M\,'} < 2$$ by definition of floating-point, we have: 

$$\begin{aligned}
& 2^{\,\mathrm{E\,'} - \,2} \cdot \frac{1.5}{1.5} \,<\, \mathrm{S} \,<\, 2^{\,\mathrm{E\,'} - \,2} \cdot \frac{2}{1.5} \\[0.3em]
\iff\; & \mathrm{E\,'} - 2 \,<\, \texttt{log2}\left( \,\mathrm{S} \,\right) \,<\, \mathrm{E\,'} - 2 + 0.415 \\[0.3em]
\implies\; & \left\lfloor\,\texttt{log2}\left(\,\mathrm{S} \,\right) \,\right\rceil \,=\, \mathrm{E\,'} - 2 \\[0.3em]
\implies\; & \mathrm{S_{Q}} \,=\, 2^{\left\lfloor\,\texttt{log2}\left(\,\mathrm{S} \,\right) \,\right\rceil} \,=\, 2^{\mathrm{E\,'} - 2} \\
\implies\; & \mathrm{B_{\text{max},\,S}} \,= \frac{\mathrm{B}_{\text{max}}}{\mathrm{S_{Q}}} =\, \frac{2^{\mathrm{E\,'}} \cdot \mathrm{M\,'}}{2^{\mathrm{E\,'} - 2}} = \, 4\,\mathrm{M\,'} \in\, (\,6,\,8\,) \\[0.3em]
\implies\; & \mathrm{B_{\text{max},\,Q}} =   \texttt{Round}\left(\,\mathrm{B_{\text{max},\,S}}\,,\, \text{FP4}\,\right) = 6
\end{aligned}
$$

The above derivation implies that: If the mantissa of $$\mathrm{B}_{\text{max}}$$ is larger than 1.5, then the scaled $$\mathrm{B}_{\text{max}}$$ will overflow outside the FP4 range<d-footnote>The above analysis can be generalized to other number representations under MX quantization: If the mantissa of B_max is larger than the mantissa of Q_max (e.g., 1.5 for FP4), then rounding (up or down) the scale factor to its nearest power-of-two will cause the scaled X<sub>max</sub> to overflow outside the representable quantization range.</d-footnote>. Consequently, the quantized $$\mathrm{B}_{\text{max}}$$ is always clamped to the largest representable FP4 value, $$6$$, leading to saturation error. On the other hand, if we only round up the scale factor to its nearest power-of-two, we have: 

$$\begin{aligned}
& 2^{\,\mathrm{E\,'} - \,2} \cdot \frac{1.5}{1.5} \,<\, \mathrm{S} \,<\, 2^{\,\mathrm{E\,'} - \,2} \cdot \frac{2}{1.5} \\[0.3em]
\iff\; & \mathrm{E\,'} - 2 \,<\, \texttt{log2}\left( \,\mathrm{S} \,\right) \,<\, \mathrm{E\,'} - 2 + 0.415 \\[0.3em]
\implies\; & \left\lceil\,\texttt{log2}\left( \,\mathrm{S} \,\right) \,\right\rceil \,=\, \mathrm{E\,'} - 1 \\[0.3em]
\implies\; & \mathrm{S_{Q}} \,=\, 2^{\left\lceil\,\texttt{log2}\left( \,\mathrm{S} \,\right) \,\right\rceil} \,=\, 2^{\mathrm{E\,'} - 1} \\
\implies\; & \mathrm{B_{\text{max},\,S}} \,= \frac{\mathrm{B}_{\text{max}}}{\mathrm{S_{Q}}} =\, \frac{2^{\mathrm{E\,'}} \cdot \mathrm{M\,'}}{2^{\mathrm{E\,'} - 1}} = \, 2\,\mathrm{M\,'} \in\, (\,3,\,4\,) \\[0.3em]
\implies\; & \mathrm{B_{\text{max},\,Q}} =   \texttt{Round}\left(\,\mathrm{B_{\text{max},\,S}}\,,\, \text{FP4}\,\right) = 3 \,\ \text{or} \,\ 4
\end{aligned}
$$

Now, $$\mathrm{B}_{\text{max}}$$ can be mapped to two representable FP4 values: $$3$$ or $$4$$, whichever is closer to the scaled $$\mathrm{B}_{\text{max}}$$. This offers more flexibility compared to the naive rounding mechanism, where the quantized $$\mathrm{B}_{\text{max}}$$ can only be mapped / clamped to $$6$$. 

To measure the algorithmic performance of MX quantization under the two scale factor rounding approaches, I implement [the naive MXFP4 quantizer](https://github.com/abdelfattah-lab/NVFP4-RaZeR/blob/5c6857a8fbbcd85e9adc47701e85992fdb1a3217/quantize/quantizer.py#L93) and [the enhanced MXFP4 quantizer](https://github.com/abdelfattah-lab/NVFP4-RaZeR/blob/5c6857a8fbbcd85e9adc47701e85992fdb1a3217/quantize/quantizer.py#L139). The following table shows the perplexity<d-footnote>Perplexity is a widely used metric to quantify the LLM performance, and <b>lower perplexity means better performance</b>.</d-footnote> of Wikitext-2 dataset on several Llama3 and Qwen3 models, under MXFP4 weight-activation quantization with a block size of 16. The round-up approach achieves much better perplexity compared to the naive round-to-nearest approach for power-of-two scale factor quantization. 
<table style="width: 80%; border-spacing: 5px; margin-left: 1em; margin-top: -1em; margin-bottom: 1.5em;"><thead>
  <tr>
    <th>  </th>
    <th> <br>Llama-3.1-8B </th>
    <th> <br>Llama-3.2-3B </th>
    <th> <br>Qwen3-4B </th>
    <th> <br>Qwen3-8B </th>
  </tr></thead>
<tbody>
  <tr>
    <td> Naive Round </td><td> 9.51 </td><td> 15.35 </td><td> 17.46 </td><td> 11.84 </td>
  </tr>
  <tr>
    <td> Round-Up </td><td> 8.69 </td><td> 13.58 </td><td> 15.69 </td><td> 10.66 </td>
  </tr>
</tbody></table>


<h3 style="margin-top: 1rem; margin-bottom: -0.25rem;">NVIDIA's (NV) Approach</h3>
Recall from Section-IV.1, if the scale factor is accurately represented (e.g., in FP32), then the maximum element can be accurately quantized without error. However, the 8-bit power-of-two scale factor used in MX violates this condition and introduces error for the block's maximum element<d-footnote>Again, the "maximum element" refers to the element with maximum magnitude.</d-footnote>. This raises the question: **Can we minimize the quantization error of block maximum while still using an 8-bit scale factor?** 

Let's try to understand the limitations of MX when quantizing the block maximum, $$\mathrm{B}_{\text{max}}$$. Again, for simplicity, assume $$\mathrm{B}_{\text{max}}$$ is a positive normal FP32 value: 

$$
\mathrm{B_{\text{max}}} = 0\;\mathrm{\underbrace{E_{7}\dots E_{0}}_{\text{Exponent}} \; \underbrace{M_{22}\dots M_{0}}_{\text{Mantissa}}} = (-1)^0 \cdot 2^{\mathrm{E}-127} \cdot (\,1+\mathrm{M}\,) = 2^{\mathrm{E\,'}} \cdot \mathrm{M\,'}
$$

Since the scale factor of MX is power-of-two, scaling $$\mathrm{B}_{\text{max}}$$ will only change its exponent without affecting mantissa:

$$
\mathrm{B_{\text{max},\,S}} \,= \frac{2^{\mathrm{E\,'}} \cdot \mathrm{M\,'}}{\mathrm{S_{MX}}} = \, 2^{\widetilde{\mathrm{E}}} \cdot \mathrm{M\,'}
$$

Comparing the above result with the case of accurate FP32 scaling: 

$$
\mathrm{S}_{\text{FP32}} = \frac{ \mathrm{B}_{\text{max}} }{ \mathrm{Q}_{\text{max}} }; \ \ \ 
\mathrm{B_{\text{max},\,S}} = \frac{ \mathrm{B}_{\text{max}}}{\mathrm{S}_{\text{FP32}}} = \mathrm{Q}_{\text{max}}
$$

We observe the accurate scaling gives $$\mathrm{B_{\text{max},\,S}} = \mathrm{Q}_{\text{max\,}}$$, which implies the mantissa of $$\mathrm{B_{\text{max},\,S}}$$ is equal to that of $$\mathrm{Q}_{\text{max}}$$ (e.g., $$1.5$$ for FP4). However, under MX scaling, the mantissa of $$\mathrm{B_{\text{max},\,S}}$$ is equal to that of $$\mathrm{B_{\text{max}}\,}$$, which can take any value in $$[\,1.0,\, 2.0\,)$$. Consequently, MX can be viewed as effectively quantizing the mantissa of $$\mathrm{B_{\text{max}}}$$ to the representable mantissas of $$\mathrm{Q}$$<d-footnote>For example, with the MXFP4 quantization described just now, B<sub>max</sub> can be mapped to 3, 4, or 6. These three quantization values have two mantissas 1.0 and 1.5, which are the representable mantissas of FP4.</d-footnote>, resulting in quantization error on $$\mathrm{B}_{\text{max}}$$. To reduce this error, one approach is to make the mantissa of $$\mathrm{B_{\text{max}}}$$ closer to that of $$\mathrm{Q}_{\text{max}}$$ through additional mantissa scaling, which requires the scale factor to also contain mantissa bits. For example, assume $$\mathrm{B}_{\text{max}}$$ has a mantissa $$\mathrm{M'} = 1.875$$. If the scale factor $$\mathrm{S}$$ can store 2-bit mantissa, then $$\mathrm{S} = 2^{\mathrm{E\,'-2}} \cdot 1.25$$ allows $$\mathrm{B_{\text{max}}}$$ to be accurately quantized to $$\mathrm{Q}_{\text{max}}$$. 

Building on this insight, NVIDIA (NV) proposes to employ FP8-E4M3--the 8-bit floating-point format with 4-bit exponent and 3-bit mantissa--for block scale quantization, as described in the following equations:  

$$\begin{aligned}
&\mathrm{S} = \frac{ |\mathrm{B}_{\text{max}}| }{ \mathrm{Q}_{\text{B,max}} } \\[0.3em] 
&\mathrm{S'} = \frac{ \mathrm{S}_{\text{max}} }{ \mathrm{Q}_{\text{S,max}} } 
= \left(\frac{ |\mathrm{B}_{\text{max}}| }{ \mathrm{Q}_{\text{B,max}}}\right)_{\text{max}} \cdot \frac{1}{\mathrm{Q}_{\text{S,max}} } = \frac{|\mathrm{B}_{\text{max}}|_{\text{max}}}{\mathrm{Q}_{\text{B,max}}} \cdot \frac{1}{\mathrm{Q}_{\text{S,max}}} = \frac{|\mathrm{T}_{\text{max}}|}{\mathrm{Q}_{\text{B,max}} \cdot\mathrm{Q}_{\text{S,max}}}  \\[0.3em] 
&\mathrm{S_\mathrm{S}} = \frac{\mathrm{S}}{\mathrm{S'}} \\[0.3em] 
&\mathrm{S_Q} = \texttt{Round}\left(\,\mathrm{S_\mathrm{S}}\,,\, \text{FP8}\,\right) \\[0.3em] 
&\mathrm{S_\mathrm{D}} = \mathrm{S_Q} \cdot \mathrm{S'}
\end{aligned}
$$

where $$\mathrm{Q}_{\text{B,max}}$$ is the maximum quantization value for the block element, $$\mathrm{Q}_{\text{S,max}}$$ is the maximum quantization value for the block scale (i.e., $$448$$ for FP8), $$\mathrm{S}'$$ is the scale of block scale, $$\mathrm{T}_{\text{max}}$$ is the tensor maximum,  $$\mathrm{S_{S}}$$ is the scaled block scale, $$\mathrm{S_{Q}}$$ is the quantized block scale, and $$\mathrm{S_{D}}$$ is the final dquantized block scale. This block scale quantization process is exactly the same as how we quantize a tensor as discussed in Section-III.2. The mathematical equations are therefore also the same as those for tensor quantization, except that we change $$\mathrm{X}$$ to $$\mathrm{S}$$, $$\mathrm{S}$$ to $$\mathrm{S'}$$, and $$\mathrm{Q}$$ to FP8. Below is a visualization to help you understand the FP8 block scale quantization using the popular NVFP4 format (i.e., $$\mathrm{Q}_{\text{B,max}}$$ = 6) as an example. 

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/NVFP4_Scale_Quant.png" width="100%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  

The above equation for calculating $\mathrm{S'}$ indicates a **tensor-wise scaling** besides block-wise scaling, as also described in [the official NVFP4 quantization blog](https://developer.nvidia.com/blog/introducing-nvfp4-for-efficient-and-accurate-low-precision-inference/)<d-footnote>Personally, I find NVIDIA’s description of tensor-wise scaling somewhat vague; hopefully, my explanation with provides greater clarity.</d-footnote>. The reason is that, when following the standard quantization procedure to quantize block scales, we meed to determine the maximum across all block scales. Since each block scale equals to its block maximum divided by $$6$$, this reduces to finding the maximum of all block maxima divided by $$6$$, which is simply the tensor maximum divided by $$6$$. 

You may also wonder: **Why choosing E4M3 instead of other FP8 variants such as E5M2 / E3M4 when quantizing the block scale?** The reason is that, empirically, E4M3 can better fit the numerical distribution of block maxima<d-footnote>According to our discussion in Section-III.2, quantizing block scales is equivalent to quantizing block maxima, since the latter differs from the former only by a constant factor (i.e., $$Q_<sub>B,max </sub>$$).</d-footnote> in LLM weight and activation tensors, resulting in lower quantization error than other FP8 variants. This brings another important topic, *Quantization Format for Tensor Elements*, which I will cover soon.

Thanks to the capability of mantissa scaling, NV reduces the quantization error of scale factors compared to MX, which in turn reduces the quantization error of block elements. To quantify these benefits, I implement [the NVFP4 quantizer](https://github.com/abdelfattah-lab/NVFP4-RaZeR/blob/5c6857a8fbbcd85e9adc47701e85992fdb1a3217/quantize/quantizer.py#L375) and compare it with [the enhanced MXFP4 quantizer](https://github.com/abdelfattah-lab/NVFP4-RaZeR/blob/5c6857a8fbbcd85e9adc47701e85992fdb1a3217/quantize/quantizer.py#L139) discussed in the last section. The following table shows the perplexity of Wikitext-2 dataset on several Llama3 and Qwen3 models, under NVFP4 and MXFP4 weight-activation quantization with a block size of 16. NVFP4 achieves much better perplexity than MXFP4. 

<table style="width: 80%; border-spacing: 5px; margin-left: 1em; margin-top: -1em; margin-bottom: 1.5em;"><thead>
  <tr>
    <th>  </th>
    <th> <br>Llama-3.1-8B </th>
    <th> <br>Llama-3.2-3B </th>
    <th> <br>Qwen3-4B </th>
    <th> <br>Qwen3-8B </th>
  </tr></thead>
<tbody>
  <tr>
    <td> Enhanced MXFP4 </td><td> 8.64 </td><td> 14.14 </td><td> 16.81 </td><td> 10.75 </td>
  </tr>
  <tr>
    <td> NVFP4 </td><td> 7.90 </td><td> 11.97 </td><td> 13.88 </td><td> 10.05 </td>
  </tr>
</tbody></table>


<h3 style="margin-top: 1rem; margin-bottom: -0.25rem;">Hardware Implication of MX and NV</h3>
The different approaches of MX and NV for block scale quantization introduce a trade-off between model accuracy and computation cost<d-footnote>Assuming both approaches use the same block size, their memory cost is roughly the same. NV requires an additional tensor-wise FP32 scale, but that single scale has negligible memory cost.</d-footnote>. Although NV achieves better model accuracy than MX, it incurs higher hardware cost to perform dequantization, as visualized below:

<div style="text-align:center;">
  <img src="/assets/img/blog/quant_codesign/MXFP4_NVFP4_Hardware.png" width="90%" />
  <figcaption style="font-size: 0.95em; margin-top: 8px;"></figcaption>
</div>  

Consider the popular FP4 block-wise quantization under a block size of 16. Let's analyze the hardware cost of performing GEMM with the two scale factor quantization approaches:
- **Block Dot-product**: Since the maximum and minimum FP4 values are $$\pm6$$ and $$\pm0.5$$, a single FP4 multiplication ranges from $$\pm0.25$$ to $$\pm36$$, which can be represented using 9-bit fixed-point. With a block size of 16, the dot-product result becomes 13 bits.
- **MX Dequantization**: To analyze the hardware cost, let's first write out the binary / decimal values of block scale and dot-product output: 

  $$\begin{aligned}
  \mathrm{S_{W}} &=\, \mathrm{E_{W}} =\, \mathrm{\underbrace{E_{7}\dots E_{0}}_{\text{Exponent}}} \,=\, 2^{\mathrm{E_{W}}-127} \\[0.3em]
  \mathrm{S_{A}} &=\, \mathrm{E_{A}} =\, \mathrm{\underbrace{E_{7}\dots E_{0}}_{\text{Exponent}}} \,=\, 2^{\mathrm{E_{A}}-127} \\[0.3em]
  \mathrm{O} \,&=\, \mathrm{S \underbrace{M_{11}\dots M_{0}}_{\text{Un-norm Mantissa}}} =\,  (-1)^{\,\mathrm{S}} \cdot \left(\,\Sigma_{j=0}^{11}\,2^{j-2}\cdot\mathrm{M}_{j} \right) 
  \end{aligned}$$

  Based on the above equations, the dequantized output should be: 
  $$
  \mathrm{O_{D}} \,=\,  (-1)^{\,\mathrm{S}} \cdot\, 2^{\mathrm{E_{W}} + \mathrm{E_{A}}-254} \cdot \left(\,\underbrace{\Sigma_{j=0}^{11}\,2^{j-2} \cdot \mathrm{M}_{j}}_{\text{Un-norm Mantissa}} \,\right)
  $$

  Surprisingly, this expression has a very similar structure to the standard 32-bit floating-point representation discussed in Section-I.2: 
  $$
  \text{FP32} \,=\, \mathrm{\underbrace{S}_{Sign} \; \underbrace{E_7\dots E_{0}}_{\text{Exponent}} \; \underbrace{M_{22}\dots M_{0}}_{\text{Mantissa}}} \,= \,(-1)^{\mathrm{S}} \cdot\, \mathrm{2^{E-127} \cdot 1.{M}}
  $$

  The dequantized output sign is the FP32 sign, the FP32 exponent $$\mathrm{E} = \mathrm{E_{W}} + \mathrm{E_{A}} - 127$$ is calculated via unsigned integer addition, and the FP32 mantissa is obtained via normalizing the dequantized output mantissa. This normalization only requires low-cost hardware components such as Leading-One Detector and Shifter. Finally, the three binary components--sign, exponent, mantissa of dequantized output--are concatenated together (via some simple wiring in hardware) to produce a FP20-E8M11<d-footnote>The dequantized output mantissa has 12 bits, but recall the floating-point representation has a hidden mantissa bit that is not stored.</d-footnote> number for cross-block partial sum accumulation. To summarize, MX dequantization has low computation cost, involving mainly unsigned integer addition, a leading-one detector, and a shifter.
- **NV Dequantization**: Again, let's write out the binary / decimal values of block scale and dot-product output: 

  $$\begin{aligned}
  \mathrm{S_{W}} &=\, \mathrm{\underbrace{E_{3}\dots E_{0}}_{\text{Exponent}} \;\underbrace{M_{2}\dots M_{0}}_{\text{Mantissa}}} \,=\, 2^{\mathrm{E_{W}}-7} \cdot 1.\mathrm{M_{W}} \\[0.3em]
  \mathrm{S_{A}} &=\, \mathrm{\underbrace{E_{3}\dots E_{0}}_{\text{Exponent}} \;\underbrace{M_{2}\dots M_{0}}_{\text{Mantissa}}} \,=\, 2^{\mathrm{E_{A}}-7} \cdot 1.\mathrm{M_{A}} \\[0.3em]
  \mathrm{O} \,&=\, \mathrm{S \underbrace{M_{11}\dots M_{0}}_{\text{Un-norm Mantissa}}} =\,  (-1)^{\mathrm{S}} \cdot \left(\,\Sigma_{j=0}^{11}\,2^{j-2}\cdot\mathrm{M}_{j} \right)
  \end{aligned}$$

  Note that the scale factor's binary pattern does not need the sign bit<d-footnote>As will be shown later, this unused sign bit can even be repurposed to further enhance the NVFP4 quantization accuracy.</d-footnote>, as it is always positive by definition of quantization. Based on the above equations, the dequantized output should be: 

  $$
  \mathrm{O_{D}} \,=\,  (-1)^{\mathrm{S}} \cdot\, 2^{\mathrm{E_{W}} + \mathrm{E_{A}}-14} \cdot \left[\,1.\mathrm{M_{W}} \cdot 1.\mathrm{M_{A}} \cdot \left(\,\Sigma_{j=0}^{11}\,2^{j-2}\cdot\mathrm{M}_{j}\,\right) \,\right]
  $$

  Similar to MX, this expression closely matches the standard floating-point representation. The dequantized output sign is the FP sign, and the FP exponent $$\mathrm{E} = \mathrm{E_{W}} + \mathrm{E_{A}} - 14$$ is calculated via unsigned integer addition. However, calculating the dequantized output mantissa is more complicated, which involves a 4-bit multiplication between the two scale factors' mantissa, followed by multiplying the 12-bit output mantissa. Then, the dequantized output mantissa is normalized to the standard FP mantissa via leading-one detector and shifter. Finally, the three binary components--sign, exponent, mantissa of dequantized output--are concatenated together to produce a FP24-E5M19 number for cross-block partial sum accumulation. 

Based on the above analysis, the NV dequantizer differs from the MX dequantizer in two notable aspects: (1) It uses a slightly cheaper adder (4-bit vs. 8-bit) for exponent addition; (2) But it requires a much more expensive $$4\text{-bit} \times 4\text{-bit} \times 12\text{-bit}$$ multiplier for mantissa calculation. Thus, the NV dequantizer consumes larger hardware area. For instance, [this paper from Meta](https://arxiv.org/abs/2603.08713) claims that NVFP4 incurs $12.6\%$ total tensor core area overhead relative to MXFP4, under the same block size of 16.




