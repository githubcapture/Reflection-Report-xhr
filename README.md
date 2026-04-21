# AAE5303 Robust Control Technology in Low-Altitude Aerial Vehicle

## Post-Lesson Reflection Report

**Student Name:** XIONG Haoran

**Student ID:** 25049841G

**Group Number:** Bug-Transporter

**Date:** 2026-04-21

---

## Section 1: AI Usage Experience

During the AAE5303 project, I used **ChatGPT** as my primary AI coding assistant, alongside occasional use of **GitHub Copilot** in VS Code. I used AI tools **3-4 times per week** during Weeks 2–12, primarily when I encountered unfamiliar technical problems or needed to understand error messages.

**What tasks did AI assist with?**
- **Debugging shell/environment issues:** In Assignment 1, I asked AI to help interpret the `Permission denied` error when accessing `/root` in WSL2, and to explain the `set +u` workaround for ROS 2 setup.bash.
- **Understanding configuration parameters:** In Assignment 2 (ORB-SLAM3), I asked AI to explain what each YAML parameter meant (e.g., `Camera.bf`, `ThDepth`, `ORBextractor.nFeatures`).
- **Writing evaluation scripts:** For the U-net project, AI helped me write the mIoU calculation function and the confusion matrix visualization code.
- **Interpreting loss curves:** When my U-net validation Dice plateaued but training loss kept decreasing, I asked AI to diagnose the overfitting symptoms.

**What features were most useful?**
- **Chat-based Q&A** was the most helpful – I could paste error messages or configuration files and get immediate explanations.
- **Code explanation** helped me understand PyTorch code snippets I found online before adapting them to my U-net pipeline.
- **Inline suggestions** (GitHub Copilot) were useful for writing repetitive data loading code, but I often had to correct coordinate conventions.

---

## Section 2: Understanding AI Limitations

This is the most important section – I encountered a clear case where AI produced incorrect advice that would have caused my model to fail if I had followed it blindly.

**What went wrong?**

During my U-net semantic segmentation work, I asked AI to help me choose a loss function for the highly imbalanced AMtown02 dataset (26 classes, with `umbrella` and `class_25` appearing in <0.1% of pixels).

The AI **confidently recommended using only Cross-Entropy loss** with class weights calculated by `sklearn.utils.class_weight.compute_class_weight`. It claimed this "should be sufficient for most segmentation tasks" and that Dice loss was "overkill for satellite imagery."

**Why did it happen?**

The AI hallucinated a general rule that did not apply to my specific dataset. It likely generalized from common advice for natural image classification (where CE with weights works well) without understanding that:
1. Aerial segmentation has extreme spatial sparsity – some classes occupy only a few hundred pixels per image
2. CE with weights still struggles when positive samples are extremely rare (the gradient signal gets drowned out)
3. Dice loss is specifically designed for this scenario because it is based on overlap, not pixel count

**How did you detect the issue?**

I detected the problem through **metric comparison**:
- I first implemented AI's recommended CE + class weights and trained for 15 epochs
- The model achieved 91% pixel accuracy but only **0.31 mIoU**
- Visual inspection showed that small objects (`umbrella`, `sedan`, `truck`) were completely ignored – the model predicted only the dominant classes (`background`, `roof`, `green_field`)

**How did you fix it?**

I searched the U-net literature and found that the original U-net paper uses **CE + Dice loss**. I implemented the combined loss:
```python
def combined_loss(pred, target, weight_ce=0.5, weight_dice=0.5):
    ce_loss = F.cross_entropy(pred, target)
    dice_loss = 1 - dice_coefficient(pred, target)
    return weight_ce * ce_loss + weight_dice * dice_loss
```
After switching to this combined loss, the mIoU improved from 0.31 to **0.619** – a 100% relative improvement. The AI's confidently wrong advice would have cost me significant time if I had not verified it against the literature.

---

## Section 3: Engineering Validation

I validated all AI-generated code and solutions using multiple independent methods, especially for the U-net segmentation pipeline.

**Testing under different conditions:**
- I tested my U-net model on **three different sequences** from the AMtown dataset (train, val, test) – not just the one used for training
- I ran inference on images with **different lighting conditions** (sunny, overcast, shadow areas) to verify robustness
- For the ORB-SLAM2 evaluation, I ran the same pipeline on **two different sequences** (HKisland_GNSS03 and a shorter test sequence) to confirm the drift pattern was consistent

**Verifying math and models:**
- For the mIoU calculation function that AI helped me write, I **manually computed IoU for one class on a small batch** (5 images) using a calculator to verify the function's output matched
- I **plotted the confusion matrix** and verified that the sum of each row equaled the total pixels for that ground-truth class (a common off-by-one bug)
- For the data preprocessing (image rescaling), I visually inspected the rescaled images and their corresponding label maps to ensure the 0.15 scaling factor did not misalign the labels

**Ensuring reliability:**
- I **compared my ORB-SLAM2 trajectory output against the ground truth** using the `evo` tool – the ATE RMSE of 2.03m gave me confidence the pipeline was working correctly
- I **cross-checked AI-suggested camera calibration parameters** against the official MARS-LVIG dataset documentation before using them
- For the environment setup (Assignment 1), I **ran the smoke tests multiple times** after each change to confirm the fix actually worked

---

## Section 4: Problem-Solving Process

**The major technical challenge** I encountered was U-net overfitting on the AMtown02 dataset – validation Dice plateaued at epoch 8 while training loss continued to decrease.

**How I approached it (step-by-step reasoning):**

1. **Initial diagnosis:** I first asked AI what could cause this pattern. The AI suggested "learning rate is too high" and "reduce model complexity." I tried both suggestions (LR 1e-5, reduced U-net depth) – neither improved validation Dice.

2. **Manual investigation:** I plotted both training and validation loss curves and noticed the gap widening after epoch 6 – a classic overfitting signature. This told me the problem was not optimizer-related, but generalization-related.

3. **Hypothesis testing:** I hypothesized that the model was memorizing training images because it saw each image only once per epoch in its original orientation (no augmentation). I implemented **random horizontal flip, vertical flip, and 90-degree rotation** augmentation.

4. **Result:** After adding augmentation, the validation Dice continued to improve up to epoch 12, and the final mIoU increased from 0.52 to **0.619**.

**How AI helped vs. failed:**
- **Failed:** AI's initial diagnosis (learning rate, model size) was incorrect – it did not recognize the overfitting pattern from the loss curves
- **Helped:** After I manually identified the cause (lack of augmentation), AI helped me write the `torchvision.transforms` code correctly and ensure the label maps were transformed identically to the RGB images

**My reasoning process:** The key insight was that the training loss kept decreasing while validation loss plateaued – this specifically indicates overfitting, not optimization issues. Augmentation is the standard remedy, so I prioritized testing that before spending time on more complex solutions (dropout, weight decay).

---

## Section 5: Learning Growth

**What skills improved the most?**
- **Deep learning for segmentation:** I went from never having trained a segmentation model to understanding U-net architecture, loss functions for imbalanced data, and mIoU evaluation in 8 weeks.
- **ROS 2 debugging:** At the start, a `Permission denied` error would have stopped me for days; now I can systematically trace permission issues, environment sourcing problems, and workspace build failures.
- **Python data pipelines:** I learned to write robust data loaders that handle image-label alignment, resizing, and augmentation correctly.

**How has my confidence changed?**
- In Week 1, the `run_smoke_tests.py` output felt like a wall of incomprehensible text. Now I can read the entire output and immediately identify what each check means and what a failure would indicate.
- At the start, I treated neural networks as black boxes. Now I can look at a loss curve and diagnose whether the problem is overfitting, underfitting, or a data issue.

**What can I do now that I couldn't before?**
- Independently set up a ROS 2 workspace, build custom packages, and debug `colcon` build errors
- Implement a complete U-net training pipeline from data loading to evaluation
- Use `evo` to evaluate SLAM trajectories and interpret ATE/RPE metrics
- Read and understand camera calibration YAML files (what each parameter does)
- Debug WSL2 permission and environment sourcing issues

---

## Section 6: Critical Reflection

**Did AI improve or hinder my learning?** Both.

**Improved:** AI accelerated my ability to write boilerplate code (data loaders, evaluation scripts, ROS launch files) and helped me understand error messages quickly. Without AI, I would have spent twice as long on environment setup and script writing, leaving less time to understand the core concepts.

**Hindered:** In Week 4, I noticed I was accepting AI-suggested code for the U-net training loop without fully understanding each line. When the model produced poor results, I could not debug because I didn't know what the code was supposed to do. I had to go back and manually rewrite the training loop from scratch to understand the gradient flow.

**Did I rely on it too much?** Appropriately after Week 5, but too much during Weeks 3–4. After realizing my shallow understanding, I set a rule: **I must explain any AI-generated code in my own words before using it** – either by adding comments or discussing it with a teammate. This forced me to engage with the code rather than passively accepting it.

**What would I do differently next time?**
1. Implement at least one core algorithm (e.g., Dice loss from scratch) before using the library version
2. Keep a log of AI prompts and responses to track what I learned vs. what I blindly accepted
3. Never use AI for coordinate transforms or calibration parameters without cross-checking the official documentation

---

## Section 7: Evidence

### 7.1 Code Snippets

**AI-suggested loss function (INCORRECT – led to poor mIoU):**
```python
# AI told me to use only weighted Cross-Entropy
class_weights = compute_class_weights(dataset)
criterion = nn.CrossEntropyLoss(weight=class_weights)
# Result: mIoU = 0.31, small objects completely ignored
```

**Corrected combined loss (from U-net literature):**
```python
# Combined CE + Dice loss – what I actually used
def dice_loss(pred, target, smooth=1.0):
    pred_soft = F.softmax(pred, dim=1)
    target_one_hot = F.one_hot(target, num_classes=26).permute(0,3,1,2).float()
    intersection = (pred_soft * target_one_hot).sum(dim=(2,3))
    union = pred_soft.sum(dim=(2,3)) + target_one_hot.sum(dim=(2,3))
    return 1 - ((2. * intersection + smooth) / (union + smooth)).mean()

loss = F.cross_entropy(pred, target) + dice_loss(pred, target)
# Result: mIoU = 0.619
```

### 7.2 Error Logs

**The Permission denied error I fixed in Assignment 1:**
```bash
$ cd /root/PolyU-AAE5303-env-smork-test
-bash: cd: /root/PolyU-AAE5303-env-smork-test: Permission denied
```
**Solution:** Copied the repository to my home directory and changed ownership:
```bash
sudo cp -a /root/PolyU-AAE5303-env-smork-test ~/
sudo chown -R $USER:$USER ~/PolyU-AAE5303-env-smork-test
```

### 7.3 Prompt Examples

**Example where AI was helpful:**
> Me: "What does the 'ThDepth' parameter mean in ORB-SLAM3's YAML config?"
> 
> AI: "ThDepth controls the minimum depth for a point to be considered 'far' vs 'near'. Points beyond ThDepth * baseline are triangulated with lower uncertainty. For monocular VO, this affects feature matching between frames."
> 
> → This was accurate and helped me tune the parameter for the UAV dataset.

**Example where AI was misleading (same as Section 2):**
> Me: "I have extreme class imbalance in my aerial segmentation dataset (some classes <0.1% of pixels). What loss function should I use?"
> 
> AI: "Use Cross-Entropy loss with class weights calculated from inverse frequency. Dice loss is overkill for satellite imagery."
> 
> → This was incorrect. For extreme sparsity, CE+weights fails; Dice or combined loss is standard.

---

**Before You Submit Checklist:**
- ✅ Deleted all blue example boxes
- ✅ Replaced guiding questions with my own writing
- ✅ Word count: ~1,150 words
- ✅ Used specific examples from my project (U-net overfitting, ORB-SLAM2 evaluation, WSL2 permission error)
- ✅ Ready to submit by GitHub Link