若矩阵$A$存在$n$个互异的特征值，这矩阵$A$一定存在$n$个线性无关的特征向量
# 差分方程

- 现在记矩阵$A$有$n$个线性无关的特征向量，记为$S = [x_1,x_2,x_3,...,x_n]$。
	则
$$
AS = 
[\lambda_1x_1,\lambda_2x_2,...,\lambda_nx_n]=
[x_1,x_2,...,x_n]
\begin{bmatrix} 
\lambda_1 & 0 & \dots & 0 \\ 
0 & \lambda_2 & \dots & 0 \\
\vdots & \vdots & \ddots & \vdots \\
0 & 0 & \dots & \lambda_n
\end{bmatrix} = 
S\Lambda
$$
- 矩阵高次幂计算
	已知$Ax=\lambda x$，则$A^k=S\lambda^{k}S^{-1}$，当$k\rightarrow\infty$时，$A\rightarrow0$当且仅当$|\lambda_i|<1$时
- 迭代推论 
	设起始向量$u_0$，$u_0=c_1x_1+c_2x_2+c_3x_3+\dots+c_nx_n$($x_1,\dots,x_n$是一组基，$c_1,\dots,c_n$是常数)
	定义$u_1 = Au_0$
	则有$u_k = A^ ku_0$
# 使用差分方程计算斐波那契数列
数列前几项为$0, 1, 1, 2, 3, 5, 8, 13, ...$
有递推关系式$F_{k+2} = F_{k+1}+F_k(F_i为数列第i项的值)$
$$
\begin{aligned}
设：\quad\quad \\ 
u_k &= [F_{k+1}, F_{k}] ^ T \\
u_{k+1} &= [F_{k+2}, F_{k+1}]^T = [F_{k+1}+F_{k}, F_k]^T\\
可以得到： \quad \\
u_{k+1} &= 
\begin{bmatrix}
1&1\\
1&0
\end{bmatrix}u_k
\end{aligned}
$$
	此时记矩阵$A=\begin{bmatrix}1&1\\1&0\end{bmatrix}$，可以快速计算数列任意一项值，如$u_{100} = [F_{101}, F_{100}]^T=A^{100} u_0$
# 微分方程
