## Summary:
1. Destroy resources after n(amount of images in swapchain + ♾) frames and
   yeet `vkDeviceWaitIdle()` out'ta here
2. Ignore `VK_SUBOPTIMAL_KHR`
3. Reuse(moving) old swapchain in the `.oldSwapchain` field
4. Do not recreate Pipeline - make viewport & scissors dynamic state
				   Renderpass - formats & requirements unlikely will change
6. Move renderer to the another thread...because of windows and...
blocking execution for any reason

# The quest for flawless Vulkan window resizing

You are following a [Vulkan](https://vkguide.dev) [tutorial](https://vulkan-tutorial.com), and just created your first window. Maybe you went ahead and rendered one whole [triangle](https://upload.wikimedia.org/wikipedia/commons/thumb/4/45/Triangle_illustration.svg/512px-Triangle_illustration.svg.png). However, at one point you accidentally dragged the border of the window and were met with a sizeable wall of cryptic text, promptly followed by an unceremonious segfault. And, if you're anything like me, this sent you on a week-long quest to not only fix the crash but create the smoothest window resize handler seen this side of the DMA channel.

... No? That's fine, you're better off learning from my adventure.

## Valiant first step - not crashing

Vulkan is not happy about what happens in your application either, and in fact gave you a warning that you need to take action *now* - the functions [`vkAcquireNextImageKHR()`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAcquireNextImageKHR.html) and [`vkQueuePresentKHR()`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueuePresentKHR.html) can return these important symbols:
- `VK_ERROR_OUT_OF_DATE_KHR`: Your swapchain is "out of date," which is standardese for "completely useless." Do not acquire any images from it from now on.
- `VK_SUBOPTIMAL_KHR`: Your swapchain is "suboptimal," meaning it can be rendered to but you will probably not like the result. Side effects include wrong aspect ratio, low resolution visuals, unacceptable performance and unhappy userbase.

When `VK_SUBOPTIMAL_KHR` is returned from [`vkAcquireNextImageKHR()`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAcquireNextImageKHR.html), we shouldn't really do anything about it - the swapchain image was still successfully acquired, and so we have an obligation to present it. At this point the swapchain should be recreated only if it is out of date, and after that is finished we can try acquiring the image again and hopefully fare better this time around. It looks more or less like this:

```c++
uint32_t swapchainImageIndex;
do { // An actual real-world example of a do-while loop? Nah, this is just a glorified goto.
    auto result = vkAcquireNextImageKHR(device, swapchain, std::numeric_limits<uint64_t>::max(),
        presentSemaphore, nullptr, &swapchainImageIndex);
    if (result == VK_ERROR_OUT_OF_DATE_KHR) {
        recreateSwapchain(); // Do the magic
        continue; // Try again
    }
    if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR)
        VK_CHECK(result); // Macro that throws or aborts if a Vulkan function returned a non-zero value
} while (false);
```

For [`vkQueuePresentKHR()`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueuePresentKHR.html), we can recreate the swapchain upon encountering either of the two conditions; the call also doesn't need to be retried. For completeness' sake, we can also trigger swapchain recreation if the size of the window is different than what we remember, just in case Vulkan missed the resize somehow.

```c++
auto result = vkQueuePresentKHR(presentQueue, &presentInfo);
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR ||
    window.size() != {swapchainExtent.width, swapchainExtent.height})
//  ^-----------^ insert your windowing system's (SDL, GLFW etc.) window size retrieval call here
    recreateSwapchain();
else if (result != VK_SUCCESS)
    VK_CHECK(result);
```

At this point, the only step remaining is to implement the `recreateSwapchain()` function. To get things working we will simply destroy and then recreate everything related to the swapchain. Before touching the swapchain, the application **must** (standardese for "has to") wait until the GPU is finished with any commands that might have been queued up in the more or less distant past - if we don't do this, we will most likely try to delete the swapchain while it is still being used and make the computer sad.

```c++
void recreateSwapchain() {
    vkDeviceWaitIdle(device); // Zzz... Self-explanatory.
    destroySwapchain();
    createSwapchain();
}
```

The functions `destroySwapchain()` and `createSwapchain()` most likely already exist in your codebase in some capacity, since you must have created the swapchain somehow and later destroyed it to prevent the validation layer from whinging. It should not be too difficult to adapt them to work in the middle of program execution. Almost all of the initial swapchain work needs to be performed again - starting from retrieving the surface formats, present modes, and capabilities. The very unlikely but *theoretically* possible case of changing the valid format implies we cannot avoid recreating any renderpass that uses a swapchain image, because image format is part of the `VkAttachmentDescription` struct.

Now, try to run your application and resize the window!

### It doesn't work!

You didn't think it would be that easy, did you? The problem is the viewport and scissor extents, which are part of a pipeline and stay the same even after our swapchain images become a different size from what we originally built the pipeline with. Does this mean we need to recreate the pipeline together with the swapchain? It is an option, but it would be much more convenient to update the pipeline with the new extent at runtime. This is enabled by making viewport and scissor part of the pipeline's *dynamic state*:

```c++
// Create an array without specifying the size. Great for C-style APIs!
auto const dynamicStates = std::to_array<VkDynamicState>({
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR,
});
auto const dynamicStateCI = VkPipelineDynamicStateCreateInfo{
    .sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO,
    .dynamicStateCount = dynamicStates.size(),
    .pDynamicStates = dynamicStates.data(),
};
// All omitted fields are value-initialized, so no issues with sNext etc.
auto const viewportStateCI = VkPipelineViewportStateCreateInfo{
    .sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO,
    .viewportCount = 1,
    .scissorCount = 1,
};
auto const pipelineCI = VkGraphicsPipelineCreateInfo{
    .sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO,
    // ...
    .pDynamicState = &dynamicStateCI,
    // ...
};
```

Notice how even though we claim to use one viewport and one scissor in the [`VkPipelineViewportStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineViewportStateCreateInfo.html) struct, the `.pViewports` and `.pScissors` pointers remain `nullptr`. They have become part of the pipeline's dynamic state, and from now on they must be inserted into the command buffer prior to a draw command:

```c++
auto const viewport = VkViewport{
    .width = static_cast<float>(swapchainExtent.width),
    .height = static_cast<float>(swapchainExtent.height),
    .minDepth = 0.0f,
    .maxDepth = 1.0f,
};
auto const scissor = VkRect2D{
    .extent = swapchainExtent,
};
vkCmdSetViewport(frame.commandBuffer, 0, 1, &viewport);
vkCmdSetScissor(frame.commandBuffer, 0, 1, &scissor);
```

Now your application will be able to continue existing after it has been resized! At least, it should. If not, try making sense of any errors you get, or let me know in the unlikely scenario that I missed something. If the render looks squished, make sure that your projection matrices are updated with the new swapchain aspect ratio.

### Is that not it?

Your prestigious cube-with-the-[awesome-face](https://www.nicepng.com/png/detail/152-1521528_awesome-face-pose-awesome-face.png)-on-it renderer is now able to change its size at runtime. The user can drag the border of the window, get a quick cup of coffee during the pipeline stall and come back to a freshly resized swapchain. If you are feeling adventurous, a small addition to the code will allow you to switch in and out of fullscreen mode. Truly, this is where any sane person would declare success and move on to more interesting tasks like, you know, [pretty-graphics](http://www.pouet.net/prod.php?which=67106) or some [actual gameplay](https://store.steampowered.com/app/753640/Outer_Wilds/). Congratulations.

&nbsp;

&nbsp;

You're still here? Brilliant. Now, let us pursue perfection.

## Stages of grief: Concurrency

You might have noticed a slight issue during a window resize on [some](https://www.microsoft.com/en-ie/windows) operating systems - as long as the user is continuously dragging the border, the application is rendering *absolutely nothing*. These operating systems do not return from message loop processing until the resize is over, blocking all of your beautiful, not copypasted code from ever getting a chance to run. The answer to this problem is, unfortunately, threads.

### What happens where

The plan is this - the main thread (the one executing `main()`) will initialize the window, spawn a second thread with a handle to this window and poll for input repeatedly until the window is closed. Or, if source code is your primary language:

```c++
auto main() {
    initializeGlfwOrSDLWhatever();
    auto window = createWindow();
    auto gameThread = std::jthread{game, std::ref(window)}; // std::jthread is nice, look it up!
    while (!window.isClosing()) {
        pollForNewEvents();
        std::this_thread::sleep_for(std::chrono::milliseconds(1)); // Prevent thread from eating up 100% CPU
    }
}
```

Meanwhile, the new thread that is spawned (let's call it the game thread) uses the window handle to create a Vulkan instance and perform rendering:

```c++
void game(Window& window) {
    VulkanEngine engine;
    engine.init(window);
    while (!window.isClosing()) {
        updateStatesOfThings();
        queueUpSomeToruses();
        engine.render();
    }
    engine.cleanup();
}
```

It is suddenly very important that you pay attention to the thread safety disclaimers of every function you run from your windowing library. To comply with the whims and fancies of [various systems](https://www.apple.com/ie/macos/)' window managers, any functions related to window setup and input handling usually must be called in the main thread. The task of implementing a thread-safe queue to pass inputs gathered on the main thread to the game thread is left as a [fun](https://dwarffortresswiki.org/index.php/DF2014:Losing) exercise to the reader.

### Looking back

What have we achieved? The code is now more complex and difficult to understand, an overlooked race condition crashes your application at startup 1% of the time, and resizing the window now feels like it just temporarily replaces your 2TB GDDR15 graphics card with a mobile SoC instead of yanking it out entirely. We are making clear progress. Next!

## Stages of grief: Moving on

It seems rather wasteful to completely recreate all swapchain storage on resize when the same memory could instead be reused. Fortunately, Vulkan presents us with a solution which, in the old tradition, we have to implement mostly manually.

The [`VkSwapchainCreateInfoKHR`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSwapchainCreateInfoKHR.html) struct has an `oldSwapchain` member, which accepts "the existing non-retired swapchain". Wait, retired? What does that mean? Furthermore, a careful reading of the spec reveals the swapchain still needs to be destroyed even though it has been passed as an `oldSwapchain`. What's the point in destroying a swapchain whose storage has been taken? Wait a moment, this all sounds familiar...

Yep, it's all a wonderfully confusing way to describe **move semantics**. The old swapchain gets *moved from*, and is put in a state that is unusable but valid. And like any other moved-from object, the "destructor call" still happens at the end. So, don't destroy the doomed swapchain right away but move it instead:

```c++
void Engine::createSwapchain(VkSwapchainKHR old = nullptr) {
    // ...
    auto swapchainCI = VkSwapchainCreateInfoKHR{
        // ...
        .oldSwapchain = std::move(old), // std::move does absolutely nothing here, but it sure makes a statement
    };
    VK_CHECK(vkCreateSwapchainKHR(device, &swapchainCI, nullptr, &swapchain));
    vkDestroySwapchainKHR(device, old, nullptr);
    // ...
}
```

Well, this was surprisingly simple. Likewise, the benefit is proportionally minuscule: you might notice more optimized VRAM usage, but our resize slideshow still struggles on with a framerate that not even a AAA studio would dare call a "cinematic experience." No more - instead of dancing around the elephant in the room, we will get to the meat of it. Mmm, elephant meat.

## Stages of grief - Reluctance

The elephant is, of course, the [`vkDeviceWaitIdle()`](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDeviceWaitIdle.html) call which we have not touched since the start of the article several PageUps ago. The pipeline stall is unacceptable, nobody likes it, it makes all the guests uncomfortable and one way or another it has to go. Awkward...

Screw it, just delete it right now. What happens? If you instinctively braced for a lashing of a lifetime, your survival instinct is working correctly because the validation layer does not take kindly to destroying resources that are still in use. Double/triple buffering techniques overlap GPU-processing of the previous frame with CPU-processing of the current frame, and it's this overlap that causes the problem - even though we are fine throwing away the current frame, the previous frame is not yet done being processed and presented.

What can we do? Driver developers HATE this one simple trick - stop destroying stuff! Don't destroy the swapchain, the framebuffers, nothing. There is no temporal conflict threatening the [fabric](https://en.wikichip.org/wiki/amd/infinity_fabric) of the digiverse, and resizing feels great! Just don't play with it too much, or the memory leak you just created catches up to the amount of VRAM inside your GPU. Forgetting that, even the validation layer doesn't mind as long as you put all the outdated objects into a virtual trash can and then destroy everything in it on app shutdown.

Do we really have to wait until shutdown to destroy the old swapchains? Not at all - if you destroyed them 10 seconds after resize, then naturally the swapchain images will be long finished being presented. What's a safe duration to wait, then? 1 second? 16.66666667 milliseconds?

The answer is not a length of time, but **exactly as many frames as your frame-overlap** (2 for double buffering, etc.)

How do we do this, then? All we need is a system for doing things *later* instead of *now*. That's right, we are teaching the computer to procrastinate.

```c++
struct DelayedOp {
    uint64_t deadline;
    std::function<void()> func;
};
uint64_t frameCounter;
std::vector<DelayedOp> delayedOps;

// Meanwhile, somewhere entirely different...
void Engine::render() {
    // ...
    auto result = vkQueuePresentKHR(presentQueue, &presentInfo);
    // ...
    
    // Run delayed ops
    auto newsize = std::ranges::remove_if(delayedOps, [this](auto& op) {
        if (op.deadline == frameCounter) {
            op.func();
            return true;
        } else {
            return false;
        }
    });
    delayedOps.erase(newsize.begin(), newsize.end()); // Erase-remove idiom

    // Advance
    frameCounter += 1;
}
```

That's it! We can now register arbitrary code to run on any frame in the future. Next, package up all your swapchain-related handles into a `Swapchain` struct - this will be a new parameter of your `destroySwapchain()` function from the first section, so that it doesn't operate on the currently live objects anymore. Given this modification, we can freely queue up a swapchain for deletion:

```c++
struct Swapchain {
    VkSwapchainKHR swapchain;
    VkFormat format;
    VkExtent2D extent = {};
    // ...
}
Swapchain swapchain; // If you aren't tired of the word "swapchain" yet, you will be now

void Engine::recreateSwapchain() {
    delayedOps.emplace_back(DelayedOp{
        .deadline = frameCounter + FramesInFlight,
        .func = [this, swapchain=swapchain]() mutable { destroySwapchain(swapchain); },
    });
    createSwapchain(swapchain.swapchain); // Still using the oldSwapchain field
}
```

Note that use of the lambda capture list is crucial here - the current state of the `Swapchain` struct is captured by copy and stored there until deletion is ready to happen. In the meantime, the original members of `swapchain` can be safely overwritten with the brand new swapchain-related objects without affecting the copy captured earlier. With that, every issue caused by removing the pipeline stall is fixed. Run your application, and then...!

## Stages of grief: Acceptance

It's over. The hours of murderous toiling have paid off. All 3 users who try to resize your application will scream in joy as they experience the unbelievable smoothness, an impossible sight, yet undeniably apparent in its glorious visage.

https://twitter.com/Tear_note/status/1349142403773571072

Well, if they're on [a certain system](https://www.linux.org) anyway.
