1. sp全量更新？
2. mmap: 页的整数倍
3. 定长编码 / 变长编码：protobuff
## MMKV三大优势
1. mmap
## 非递归锁
1. 不可重入锁
2. 设置可重入的属性

# 写
1. MMKV::doAppendDataWithKey
	1. MMKV::loadFromFile 初始化CodedOutputData m_output 记录mmap的指针 ptr
		1. m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);