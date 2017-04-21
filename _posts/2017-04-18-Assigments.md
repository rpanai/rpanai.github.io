---
layout: post
title: Generate random assignments/exams in LaTeX with SageMath and Python
---
It returns one pdf with questions and one with solutions for each student. Here we generate a Linear Algebra test.


**I don't want to read. Just show me the code in [SMC](https://cloud.sagemath.com/projects/49d1983f-3271-414e-9103-6c355446c247/files/Assignments%20Generator/) or [GitHub](https://github.com/rpanai/SageMathNotebooks/tree/master/Assignments%20Generator)!**


**WARNING: If you download it from github please note that the file `SageMath_notebook.ipynb` is a SageMath jupyter notebook!**

First of all something about the title: [SageMath](http://www.sagemath.org/) is written in Python and handles LaTeX. So the title is surely redundant. I strongly suggest to give a try to the cloud version of SageMath [SMC](https://cloud.sagemath.com) if you want to have an idea of how it works. If you are not a Linux's user the cloud version is the only alternative to a VM. Furthermore the cloud version comes with a full LaTex editor, Jupyter notebooks, Linux terminal and a Course Management. 


<!--Post structure:

1. [Generate problems](#generate-problems)
2. [Generate LaTeX assignments with Jinja2](#generate-latex-assignments-with-jinja2)
-->

# 0. Preamble

When I first used SageMath to generate assignments let's say that my code wasn't very elegant. 
Few days ago I tried to use the package `sagetex` inside a *.tex* file following Al-Safi's [paper](https://www.researchgate.net/publication/295076278_Randomising_assignments_with_SageTex). The advantage is that you have everything inside a single file, the disvantages are:
 
* you need to compile every single assignment (and markscheme) $$3$$ times.
* you need to prepare your exercises on SageMath first and formatting your solution in a particular way may not work with `sagetex`.
* on SMC `sagetex` runs flawless but the configuration on a local machine can be quite tedious.

So to compile 100 copies of a simplified version of the assignment below took me more than five minutes while using jinja2 took about $$40$$ seconds [see](#benchmark).

Despite the two solutions are differents (one single file versus several) we can convert one to the other with `pdftk`.



# 1. Generate problems

In some sense this is the most interesting part as you don't want to have assigments with different levels of difficulty. In this case we produce the exercise and its solution but, with some more coding, it is possible to generate the step-by-step solution.

## 1.1 Calculate the determinant

The matrix $$A$$ generated satifies the following conditions:

* All entries $$a_{ij}\in \mathbb{N}$$ and $$6\leq a_{ij}\leq 12$$
* The absolute value of the determinant is bounded $$100 < \vert \det(A)\vert <150$$

The following code is self-explanatory.

{% highlight python %}
def generate_det(n,lb=6,ub=12,dlb=100,dub=150):
    control=0
    while control==0:
        A=random_matrix(ZZ, n,x=lb,y=ub);
        d=det(A)
        if dlb<abs(d)<dub:
            control=1
    return(A,d)
{% endhighlight %}



## 1.2 Find the inverse

The matrix $$A$$ generated satifies the following conditions:

* All entries $$a_{ij}\in \mathbb{N}$$ and $$1\leq a_{ij}\leq 5$$
* The absolute value of the determinant is bounded $$2 < \vert \det(A)\vert <8$$
* All the entries of $$A^{-1}$$ are in $$\mathbb{Q}\setminus\mathbb{Z}$$

In particular the last point implies that $$0\notin A^{-1}$$.

{% highlight python %}
def generate_inverse(n,lb=1,ub=5,det_min=2,det_max=8):
    control=0
    while control==0:
        A=random_matrix(ZZ, n,x=lb,y=ub);
        control=A.determinant()
        if control!=0 and det_min<abs(control)<det_max:
            B=A.inverse()
            if any(x in ZZ for x in B.list()):
                control=0
        else:
            control=0
    return(A,B)
{% endhighlight %}

If you are not familiar with Python the only part to comment is
{% highlight python %}
if any(x in ZZ for x in B.list()):
    control=0
{% endhighlight %}

which use the so-called *list of comprehension* and produce the same result of the following code
{% highlight python %}
for x in B.list():
    if x in ZZ:
        control=0
        break
{% endhighlight %}

## 1.3 Eigenvalues and Eigenvectors

This is a typical exercise. The condition for the matrix $$P$$ are:

* All entries $$a_{ij}$$ are in $$\mathbb{Z}\setminus\{0\}$$ and $$2\leq\vert a_{ij}\vert\leq 10$$
* All eigenvalues $$\lambda$$ are in $$\mathbb{Z}$$ and $$2 <\lambda <8$$

{% highlight python %}
def generate_eigenvalues(n,lst):
    if len(lst)<n:
        print("ERROR: insert a bigger list!")
    else:
        shuffle(lst)
        return([choice([-1,1])*el for el in lst[:n]])

def diagonalizable(n,lst=range(2,8),lb=2,ub=10):
    L=diagonal_matrix(generate_eigenvalues(n,lst))
    control=0
    while control==0:
        Q=random_matrix(ZZ, n,x=-4,y=4) # you can modify this
        control=det(Q)
        if control!=0:
            P=Q*L*Q.inverse()
            plst=P.list()
            # we want an integer matrix without 0s
            if not all(x in ZZ for x in plst) or 0 in plst:
                control=0
            plst=[abs(x) for x in plst]
            # for every entry x in P lb<=abs(x)<=ub
            if min(plst)<lb or max(plst)>ub:
                control=0
    var('l', latex_name='\lambda')
    d=str(latex(((C-l*identity_matrix(n)).det()).expand()))
    values=', '.join([str(latex(l))+"_"+str(idx+1)+"="+str(val) 
                      for idx, val in enumerate(P.eigenvalues())])
    vecs=', '.join(["v_{"+str(el[0])+"}="+str(latex(matrix(el[1]).transpose())) 
                    for el in P.eigenvectors_right()])
    return(P,d,values,vecs)
{% endhighlight %}

# 2. Generate assignments in LaTeX with Jinja2

Basically in my setup I have an empty latex document *questions.tex* 

{% highlight latex %}
\documentclass[10pt]{exam}
\usepackage{nopageno}
\usepackage{filecontents, amsmath,amsfonts}
\usepackage{enumerate}
\usepackage{nicefrac}

\renewcommand{\frac}[2]{\nicefrac{#1}{#2}}

\begin{document}
%\printanswers
\input{assignments.tex}
\end{document}
{% endhighlight %}

and another called *solutions.tex*

{% highlight latex %}
\documentclass[10pt]{exam}
\usepackage{nopageno}
\usepackage{filecontents, amsmath,amsfonts}
\usepackage{enumerate}
\usepackage{nicefrac}

\renewcommand{\frac}[2]{\nicefrac{#1}{#2}}

\begin{document}
\printanswers
\input{assignments.tex}
\end{document}
{% endhighlight %}

As you can see, the only difference is that on the second file `\printanswers` is not commented. 
In both case *assignments.tex* contains all the juice and it is generated by jinja2 from the following *template.tex*


{% highlight latex %}

\marginpar{cod.\VAR{code}}
\begin{center}
\fbox{\fbox{\parbox{5.5in}{\centering
Linear Algebra - Assignment 1\\
Prof. Roberto Panai}}}
\end{center}

\begin{questions}

\question Calculate the determinant of A, where
$$
A=\VAR{A}
$$

\begin{solution}
$$\det(A)=\VAR{d}$$
\end{solution}

\question Find the inverse of $A$, if it exists, using the Gauss-Jordan elimination. Where
$$
A=\VAR{B}
$$

\begin{solution}
$$A^{-1}=\VAR{B_inv}$$
\end{solution}

\question Let $T:\mathbb{R}^3\rightarrow\mathbb{R}^3$  a linear map and
 
$$
A=\VAR{C}
$$
the matrix of $T$ with respect to the canonical basis.
\begin{parts}
\part Calculate $\det(A-\lambda I)$.
\part Find the eigenvalues for $A$.
\part Find the eigenvectors for $A$.
\part Is $A$ diagonalizable?
\end{parts}
\begin{solution}
\begin{enumerate}[(a)]
\item $\det(A-\lambda I)=\VAR{dl}$.
\item $\VAR{values}$.
\item $\VAR{vecs}$.
\item Yes.
\end{enumerate}
\end{solution}

\end{questions}

\newpage


{% endhighlight %}

## 2.1. Run it!

The main problem of random generated assignments is that whenever you generate them again the exercises are different. If you are not carefully enough you can lost all the markschemes. To avoid this we can `set_random_seed()` for every assignment. The idea is to use the student id number for it. From a list of students is not particularly hard to extract the id numbers and put them on a *student_numbers.csv* file, where every line contains one student id. 

For a demo we can generate them with
{% highlight python %}
with open("student_numbers.csv","w") as f:
    f.write("\n".join(str(randint(100000,999999)) for i in range(100)))
{% endhighlight %}



{% highlight python %}
import jinja2
import subprocess

def generate_jinja():
        
    latex_jinja_env = jinja2.Environment(
                    block_start_string = '\BLOCK{',
                    block_end_string = '}',
                    variable_start_string = '\VAR{',
                    variable_end_string = '}',
                    comment_start_string = '\#{',
                    comment_end_string = '}',
                    line_statement_prefix = '%%',
                    line_comment_prefix = '%#',
                    trim_blocks = True,
                    autoescape = False,
                    loader = jinja2.FileSystemLoader(os.path.abspath('.'))
                    )
    
   f=open('assignments.tex','w')
    for idx, seed in enumerate(open('student_numbers.csv').read().split('\n')):
        set_random_seed(int(seed))
        (A,d)=generate_det(4)
        (B,B_inv)=generate_inverse(3)
        (C,dl,values,vecs)=diagonalizable(3)
    
        diz={'code':'{:03}'.format(idx+1),
             'A':str(latex(A)),
             'd':str(d),
             'B':str(latex(B)),
             'B_inv':str(latex(B_inv)),
             'C':str(latex(C)),
             'dl':dl,
             'values': values,
             'vecs': vecs
             }
        template = latex_jinja_env.get_template('template.tex')
        text=template.render(diz)
        f.write(text.encode("UTF-8"))
    f.close()
    subprocess.call(['pdflatex', 'questions.tex'])
    subprocess.call(['pdflatex', 'solutions.tex'])
    folder="PDF"
    if not os.path.exists(folder):
        os.mkdir(folder)
    os.rename('questions.pdf',os.path.join(folder,"questions.pdf"))
    os.rename('solutions.pdf',os.path.join(folder,"solutions.pdf"))
    os.chdir('..')
{% endhighlight %}

The *latex_jinja_env* comes from [here](http://eosrei.net/articles/2015/11/latex-templates-python-and-jinja2-generate-pdfs).

If you don't want to use the student id numbers to set the random seed you can change these lines 

{% highlight python %}
for idx, seed in enumerate(open('student_numbers.csv').read().split('\n')):
        set_random_seed(int(seed))
{% endhighlight %}

with

{% highlight python %}
# generate 100 assigments
for idx in range(100):
{% endhighlight %}


## 2.2 Benchmark

Compile two pdfs (questions and solutions) with 100 assignemnts each on my laptop

{% highlight python %}
%time generate_jinja()
{% endhighlight %}

took

{% highlight raw %}
CPU times: user 37.3 s, sys: 624 ms, total: 37.9 s
Wall time: 38.7 s
{% endhighlight %}




 

