#教你写一个hash函数

参考[Fowler–Noll–Vo hash function](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function#FNV-1a_hash)
这个hash函数简称FNV，是一种碰撞率较低的**非加密**的hash算法，它最突出的优势是它非常易于实现。
目前主要使用的有FNV-1和FNV-1a两种，这两种差异非常小：

    FNV-1 hash
    
       hash = FNV_offset_basis
       for each byte_of_data to be hashed
       	hash = hash × FNV_prime
       	hash = hash XOR byte_of_data
       return hash

    FNV-1a hash
    
       hash = FNV_offset_basis
       for each byte_of_data to be hashed
       	hash = hash XOR byte_of_data
       	hash = hash × FNV_prime
       return hash
是不是非常简单明了呢？注意其中有两个固定数：
    
     FNV_offset_basis= 14695981039346656037 
     FNV_prime= 1099511628211
为啥是这两个？因为使用它们两个时碰撞概率最小。

由此我们自己写出了一个FNV-1a的hash函数：

    const offset int64 = 14695981039346656037
    const prime int64 = 1099511628211
    
    // FNV-1a 
    func Hash(b []byte) int64 {
    	res := offset
    	for _,b := range b{
    		res ^= int64(b)
    		res *= prime
    	}
    	return res
    }
是不是非常简单呢？