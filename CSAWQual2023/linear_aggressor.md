#  Linear Aggressor

### Description

> Wall Street Traders dropped a new model! I hope no one can steal it.

### Solution

This challenge caught my eye because I realized the name hinted at linear regression, a modeling technique and ML algorithm I learned about while taking a Python for data science class. Connecting to the challenge, I saw that it took 30 numbers as input, one-by-one, and produced a single number. Trying something naive, like all 1s produced 2809.

![linear.png](../images/linear.png) 

While I didn't remember every detail, I recalled learning that a linear regression model was basically a weighted sum of variables something like $a_1x_1 + a_2x_2 + ... + a_nx_n$. If that's all this program is doing, one can then "steal" the model by finding the weights, and these weights probably correspond to the flag. Since this equation is linear, I should be able to solve for each weight iteratively by setting its associated variable to 1 and all others to 0. The result should then be weight itself (see note at end).

To test my assumptions, I entered a 1 as the first value followed by 29 0s and obtained the result 224. I reconnected and repeated this sequence to confirm the result did not change. Next, I entered a 0, then 1, and 28 more 0s, producing 240. Since I knew the flag format prefix was `csawctf{`, I compared the distance between the decimal ASCII values of `'c'` and `'s'` to the difference between 224 and 240. Since these difference were both 16, it seemed like I was on the right track. I then prepared my solution script.

```python
from pwn import *
import time

weights = []
flag = ""

for i in range (30):
	p = remote("misc.csaw.io", 3000)
	r = p.recv()
	# print(r)
	print("Trying value 1 at position %d" % i)
	for j in range(30):
		p.sendline(bytes(str(int(j == i)), 'utf-8'))
		r = p.recvline(keepends=False)
		time.sleep(.01)
	r = p.recvline()
	r = int(r.decode('utf-8').rstrip())
	#print(r)
	print(f"Weight at position {i}: {r}")
	weights.append(r)
	print("Model weights:", weights)
	p.close()

offset = weights[0] - ord('c')
flag = bytearray([w - offset for w in weights]).decode('ascii')
print("\n",flag, "\n")
```

After 30 iterations of the outer loop, the script has all 30 weights. It then converts the weights to an ASCII string by calculating and subtracting the offset from the suspected first character of the flag. As suspected the model weights yield the flag.

![linear_flag.png](../images/linear_flag.png) 

### Note

One thing I neglected in retrospect is that the weighted sum also has a constant term, which is the offset I found by knowing the flag format above. So the model would actually look more like $a_0 + a_1x_1 + a_2x_2 + ... + a_nx_n$. Without the flag format information, I could have still obtained the constant term $a_0$ by entering all 0s in the model, which results in 125, which is the same offset subtracted in my solution above.