[gmic_filter_development_guide.md](https://github.com/user-attachments/files/22673648/gmic_filter_development_guide.md)
# G'MIC Filter Development Guide
## Based on Development of Dream Smoothing Enhanced Filter
**Created: October 2025 | Source: Collaborative development with Claude Sonnet**

---

## Table of Contents
1. [Critical Syntax Rules](#critical-syntax-rules)
2. [Working Command Patterns](#working-command-patterns)
3. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
4. [Verified G'MIC Commands](#verified-gmic-commands)
5. [Image Stack Management](#image-stack-management)
6. [Performance Optimization](#performance-optimization)
7. [Working Filter Template](#working-filter-template)
8. [Debugging Approach](#debugging-approach)

---

## Critical Syntax Rules

### Blend Command
**CORRECT:**
```gmic
blend mode,opacity,reverse_order
blend alpha,0.8,0
blend screen,0.5,0
```

**INCORRECT (causes stack errors):**
```gmic
blend[-1,-2] mode,opacity
blend[0] [1],mode,opacity
add[-2] [-1],opacity,0
```

### Duplication and Processing Pattern
**CORRECT:**
```gmic
+command              # Duplicate then apply command to copy
blur. 5               # Process the duplicate (. means last image)
blend mode,opacity,0  # Blend back with original
```

**INCORRECT:**
```gmic
+                     # Just duplicate
command[1]            # Try to specify which image
blend[-1,-2] mode     # Explicit indices fail
```

### Variable Usage in Expressions
**CORRECT:**
```gmic
fog_val={$fog*2}
add $fog_val
c 0,255
```

**INCORRECT:**
```gmic
f "i+($fog*2)"        # Variable in quotes fails
f "i+{$fog*2}"        # Curly braces in quotes fail
```

---

## Working Command Patterns

### The Core Dream Smoothing Algorithm
From the original Dream Smoothing by Arto Huotari:

```gmic
repeat $iterations
  +fx_smooth_anisotropic {$amplitude/$iterations},0.4,0.5,0.6,2,0.8,30,2,0,0,1,0,1,24,0
  fx_smooth_anisotropic. {$amplitude*1.4/$iterations},0.4,1,0.6,4,0.8,15,5,0,1,1,0,1,24,0
  blend alpha,$opacity,0
done
```

**Key Points:**
- Two passes of anisotropic smoothing with different parameters
- Division by iterations to maintain consistent effect
- Blend with alpha mode to preserve detail while adding dreaminess
- The `.` operator applies command to last image in stack

### Effect Application Pattern
```gmic
if $parameter>threshold
  +command parameters     # Create effect layer
  n. 0,255               # Normalize the effect
  blend mode,strength,0  # Blend with main image
fi
```

### Safe Quantization (Color Reduction)
```gmic
if $watercolor>2                          # Check threshold first
  +quantize {max(16,128-$watercolor*5)},0,1
  blend normal,{$watercolor/18},0
fi
```

---

## Common Mistakes to Avoid

### 1. Non-Existent Commands
**DO NOT USE:**
- `oilpaint` → Use `kuwahara` instead
- `posterize` → Use `quantize` instead  
- `pixelize` → Use manual `resize` operations
- `emboss` → Use `gradient_norm` or convolution

### 2. Wrong blend Syntax
**WRONG:**
```gmic
blend[dest] [source],mode,opacity
blend[-2,-1] mode,opacity
```

**RIGHT:**
```gmic
blend mode,opacity,0
```

### 3. Complex Stack Management
**AVOID:**
```gmic
store varname
restore varname
remove[-3,-2]
keep[indices]
```

**PREFER:**
```gmic
+command              # Natural duplication
blend mode,opacity,0  # Simple blending
# Let G'MIC manage the stack
```

### 4. Assuming Parameter Strength
**PROBLEM:** Parameters in 0-1 range are often too subtle
**SOLUTION:** Use ranges like 0-100, 0-15, etc., then scale internally
```gmic
# User sees: Glow = 50 (0-100 range)
# Code uses: blur {$glow/2}, blend screen,{$glow/250},0
```

---

## Verified G'MIC Commands

### Smoothing & Filtering
- `fx_smooth_anisotropic` - Anisotropic diffusion (THE dream smoothing core)
- `bilateral` - Edge-preserving smoothing
- `blur` - Gaussian blur
- `kuwahara` - Median filter (oil painting effect)
- `smooth` - Anisotropic smoothing (simpler than fx_smooth_anisotropic)

### Artistic Effects
- `kuwahara size` - Creates painterly effect
- `quantize colors,dithering,keep` - Color reduction
- `gradient_norm` - Edge detection
- `threshold percentage%` - Binary threshold

### Color Adjustments
**CORRECT SYNTAX:**
```gmic
adjust_colors brightness,contrast,gamma,hue_shift,saturation
adjust_colors 10,5,0,0,15  # Brightness +10, Contrast +5, Sat +15
```

### Blending Modes (Verified Working)
- `alpha` - Opacity blend (preserves detail)
- `screen` - Brightening (for glow effects)
- `multiply` - Darkening (for vignette)
- `overlay` - Contrast blend
- `softlight` - Subtle contrast
- `normal` - Standard blend

### Utility Commands
- `n min,max` - Normalize (alias for `normalize`)
- `c min,max` - Cut/clamp (alias for `cut`)
- `to_color` - Force RGB
- `to_gray` - Convert to grayscale
- `to_rgb` - Ensure RGB format
- `resize2dx width,interpolation` - Smart resize by width

---

## Image Stack Management

### How the Stack Works
```gmic
# Start: [0]=original image
+blur 5              # [0]=original, [1]=blurred copy
n. 0,255            # Normalize [1] (last image)
blend screen,0.5,0  # Blend [1] into [0], removes [1]
# End: [0]=result
```

### The Dot Operator
- `.` = Last image in stack
- `..` = Second to last
- `...` = Third to last

### Safe Pattern for Effects
```gmic
# This pattern always works:
+effect_command      # Creates copy and applies effect
process_effect.      # Process the effect layer
blend mode,amt,0     # Automatically blends and cleans up
```

---

## Performance Optimization

### Quality Levels
```gmic
# Draft quality (fast preview)
if $quality==0
  orig_w={w}
  resize2dx 50%,2
  # ... processing ...
  resize2dx $orig_w,5
fi
```

### Speed Tips for Users
1. **Reduce Iterations:** 1-2 instead of 3-5 (linear speedup)
2. **Lower Amplitude:** 200-300 instead of 430+ (processing intensity)
3. **Preview at smaller size:** Built into most filters
4. **Disable unused effects:** Each effect adds processing time

### Why fx_smooth_anisotropic is Slow
- Performs tensor calculations for edge detection
- Anisotropic diffusion is computationally expensive
- Multiple iterations compound the cost
- No way around it - this is the algorithm's nature

---

## Working Filter Template

```gmic
#@gmic
#
#  Filter Name
#  Author attribution
#  License
#

#@gui _<b>Category</b>
#@gui Filter Name : fx_filter_name, fx_filter_name_preview(1)
#@gui : Parameter 1 = int(3,1,10)
#@gui : Parameter 2 = float(50,0,100)
#@gui : sep = separator()
#@gui : Preview Type = choice("Full","Forward Horizontal","Forward Vertical")
#@gui : Preview Split = point(50,50,0,0,200,200,200,0,10)_0

fx_filter_name :
  to_color
  
  param1=$1
  param2=$2
  
  # Main processing
  repeat $param1
    +fx_smooth_anisotropic parameters
    blend alpha,0.8,0
  done
  
  # Additional effects
  if $param2>5
    +blur {$param2/2}
    blend screen,{$param2/200},0
  fi
  
  n 0,255
  c 0,255

fx_filter_name_preview :
  w_orig={w}
  if $w_orig>800
    resize2dx 50%,2
  fi
  fx_filter_name $*
  if w<$w_orig
    resize2dx $w_orig,5
  fi
```

---

## Debugging Approach

### Step 1: Start Minimal
```gmic
fx_test :
  to_color
  blur 5
  n 0,255
```
Test basic functionality first.

### Step 2: Add One Feature at a Time
```gmic
fx_test :
  to_color
  +blur 5              # Add duplication
  blend alpha,0.5,0    # Add blending
  n 0,255
```
Test each addition separately.

### Step 3: Use Echo for Debugging
```gmic
echo "Parameter value: "$param
echo "Processing step 1"
```
Remove once working.

### Step 4: Test Command Line First
```bash
gmic input.jpg blur 5 n 0,255 -o test.jpg
```
Verify commands work before adding to filter.

### Common Error Messages

**"Invalid selection [-1]"**
- You're using explicit stack indices
- Solution: Remove indices, use simple `blend mode,opacity,0`

**"Unknown command 'commandname'"**
- Command doesn't exist in G'MIC
- Solution: Check verified commands list, find alternative

**"contains index X, not in range"**
- Stack management error
- Solution: Simplify, let G'MIC handle the stack automatically

---

## Real-World Example: Dream Smoothing Enhanced

**Project Context:**
- Based on original Dream Smoothing by Arto Huotari
- Goal: Add artistic effects while maintaining core algorithm
- Challenge: Speed optimization for large images (4000x3000px)

**What Worked:**
```gmic
repeat $iterations
  +fx_smooth_anisotropic {$amplitude/$iterations},0.4,0.5,0.6,2,0.8,30,2,0,0,1,0,1,24,0
  fx_smooth_anisotropic. {$amplitude*1.4/$iterations},0.4,1,0.6,4,0.8,15,5,0,1,1,0,1,24,0
  blend alpha,$opacity,0
done
```

**Effects Layer Pattern:**
```gmic
# Glow effect
if $glow>5
  +blur {$glow/2}
  n. 0,255
  blend screen,{$glow/250},0
fi

# Grain effect (fine, monochrome)
if $grain>5
  +noise {$grain/20}%,0
  blur. 0.3
  to_gray.
  to_rgb.
  blend overlay,{$grain/200},0
fi
```

**Processing Time:**
- 4000x3000px, 3 iterations, amplitude 430: ~90 seconds
- Optimization: Expose all parameters for user control
- No significant speedup possible without changing algorithm

---

## Best Practices Summary

1. **Use simple blend syntax** - `blend mode,opacity,0`
2. **Let G'MIC manage the stack** - Don't use explicit indices
3. **Test commands individually** - Verify they exist and work
4. **Use strong parameter ranges** - 0-100 instead of 0-1
5. **Add effects incrementally** - Test each addition
6. **Study working filters** - Learn from Arto Huotari's patterns
7. **Document parameter purposes** - Help future maintenance
8. **Normalize/clamp at the end** - `n 0,255` then `c 0,255`

---

## Reference: Original Dream Smoothing Core

From `arto_huotari.gmic`:
```gmic
repeat $Iterations
  # Resize (for progressive refinement)
  IWidth={0,round(w/($<+1))}
  IHeight={0,round(h/($<+1))}
  if $>!=0
    r. $IWidth,$IHeight,1,3,5,1
  fi
  
  +r[0] $IWidth,$IHeight,1,3,5,1
  
  # Two-pass anisotropic smoothing
  fx_smooth_anisotropic. {430/$Iterations*($<+1)},0.4,0.5,0.6,2,0.8,30,2,0,0,1,0,$Threads,$Overlap,0
  fx_smooth_anisotropic. {600/$Iterations*($<+1)},0.4,1,0.6,4,0.8,15,5,0,1,1,0,$Threads,$Overlap,0
  
  # Blend with previous iteration
  if $>!=0
    blend[-1,-2] ${_mode},${Opacity},${ReverseOrder}
    if $Eqa
      equalize. 256
    fi
  fi
done
```

---

## Limitations & Constraints

### What G'MIC Filters Cannot Do
- Interactive buttons (reset, save preset, etc.)
- Dynamic preset saving from GUI
- Reorder/favorite presets programmatically
- Access to system storage for settings
- Complex UI interactions

### What Can Be Done Instead
- Hardcoded preset dropdown with predefined values
- User manual note-taking of favorite settings
- G'MIC-Qt may have built-in settings save feature
- Comments/documentation in the code

### GUI Parameter Limitations
- Parameters are declarative only
- No conditional UI elements
- No grouping/collapsing sections dynamically
- Layout is linear/sequential only

---

## Final Notes

This guide represents lessons learned from actual filter development, debugging real errors, and studying working G'MIC filters. The patterns documented here are proven to work in production.

**Key Success Factor:** Study existing working filters (like those by Arto Huotari) and follow their patterns exactly before attempting modifications.

**When Stuck:** Strip back to minimal working code, then add features one at a time while testing each addition.

**Remember:** G'MIC's strength is in its powerful image processing algorithms, not in UI flexibility. Focus on the processing logic and let the G'MIC-Qt interface handle presentation.

---

**Document Version:** 1.0  
**Created:** October 2025  
**Based On:** Dream Smoothing Enhanced development session  
**Designers:** Zarir Madon (filter design), Claude Sonnet (implementation)  
**License:** This documentation is provided as-is for educational purposes

---

## Appendix: Quick Command Reference

```gmic
# Smoothing
bilateral spatial,range,sampling_s,sampling_r
fx_smooth_anisotropic amplitude,sharp,aniso,alpha,sigma,dl,da,prec,interp,fast,tile,parallel,spatial_overlap,threads
blur radius
kuwahara size

# Color
adjust_colors brightness,contrast,gamma,hue,saturation
to_color
to_gray
to_rgb

# Effects
gradient_norm smoothness,linearity,min_threshold,max_threshold,negative
quantize colors,dithering,keep_colors
threshold percentage%

# Blending
blend mode,opacity,reverse
# Modes: alpha, screen, multiply, overlay, softlight, normal

# Utility
n min,max           # Normalize
c min,max           # Cut/clamp
+ or +command       # Duplicate (then apply command)
.                   # Last image
resize2dx width,interpolation
echo "message"      # Debug output
```
