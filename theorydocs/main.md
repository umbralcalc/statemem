### Theory documentation for [${\tt pneumoinfer}$](https://github.com/umbralcalc/pneumoinfer)

---

#### Motivating background

Multi-state models - stochastic processes occupying one of a finite set of states at each moment in time - appear to describe many natural phenomena, but are probably most frequently used in the mathematical modelling of population health~\cite{anderson1992infectious, daley1999epidemic, hougaard1999multi, commenges1999multi}. The statistical inference (or selection) of these models for real-world applications frequently involves data in the form of a sequence of individual state observations, which are often coupled with some diagnostic uncertainty~\cite{hougaard2000statistical}.

There are over 90 known capsular serotypes of _Streptococcus pneumoniae_, which persist despite their mutual competition for the same ecological niche (the nasopharynx) and a known fitness gradient. Motivated by the global pneumococcal disease burden~\cite{watt2009burden}, a specific class of multi-state models has been developed to describe the carriage dynamics which offers a neat explanation of this persistence through immunity-driven stabilisation effects~\cite{cobey2012niche}. This class of model typically uses a counting memory of past state (or serotype) occupations (or colonisations) as a model for human immunity (see, e.g., Ref.~\cite{flasche2013impact} for an alternative formulation and Ref.~\cite{lochen2020dynamic} for a review of transmission models). Building from these mathematical models, a range of statistical approaches have also been used to infer the pneumococcal carriage through a given population from nasopharyngeal swab sample data~\cite{lipsitch2012estimating,numminen2013estimating,cauchemez2006investigating}. All of this is really important, e.g., to understanding more precisely how a vaccine covering a restricted range of serotypes can impact colonisation in a given community or region~\cite{cai2018use}. 

The design of policies for gathering data will always have a direct impact on the quality and utility of information that can be learned about a model via statistical inference. Therefore, it's typically useful to know \emph{a priori} the fundamental constraints a given policy might impose on this procedure. The purpose of the \href{https://github.com/umbralcalc/pneumoinfer}{\texttt{pneumoinfer}} class and its accompanying \href{https://github.com/umbralcalc/pneumoinfer/tree/master/notebooks}{\texttt{Jupyter notebooks}} is to provide researchers with a rigorous framework to investigate these limitations for the inference of multi-state models with a counting memory --- which are structurally inspired by the pneumococcus carriage models of Refs.~\cite{cobey2012niche,lipsitch2012estimating}. 

In this documentation, we're going to: analyse the master equation of a stochastic model which includes memory effects from individual immunity; investigate a novel (to our knowledge) approximate ODE description for the dynamics, while assessing its validity; and utilise a Fisher approximation to study the effect of varying the total time period of sampling, the sample size and the number of states to establish fundamental bounds on inferring parameters of the model. By then exploiting the new efficient ODE description, we will be able to develop a new method of inference that avoids biasing from overly large sampling windows; which is the main inference method implemented in the \href{https://github.com/umbralcalc/pneumoinfer}{\texttt{pneumoinfer}} class. All of the figures and tools for the analysis are generated in the \href{https://github.com/umbralcalc/pneumoinfer/tree/master/notebooks}{\texttt{Jupyter notebooks}} of the public repository. Hence, the reader should be able to follow along section-by-section quite easily.

---

#### Model analysis

##### Fixed $\Lambda_i$ model

Inspired by the pneumococcus carriage models introduced in Refs.~\cite{cobey2012niche,lipsitch2012estimating}, let's now construct a multi-state model which incorporates a counting memory of past state occupations. This will include: an event rate of state clearance $\tilde{\mu}_i$ --- the rate at which an individual occupying the $i$-th indexed state returns to the null state; an event rate of susceptibility $\tilde{\Lambda}_i$ for an individual moving to the $i$-th state from the null state; and a state-specific factor matrix $f_{ii'}$ which rescales $\tilde{\Lambda}_{i'}$ to create an event rate for an individual moving directly between the $i$-th and $i'$-th states. 

Now consider $\tilde{\mu}_i=\tilde{\mu}_i( \dots, n_{i}, \dots )$, i.e., a function of all previous state occupations by the individual, where $n_i$ are the state-specific counts of past occupations. The rate $\tilde{\mu}_i$ hence maintains a `record' of past state occupations and updates accordingly through this memory. Additionally, we will make each rate $\tilde{\Lambda}_i=\tilde{\Lambda}_i(n_{i})$, i.e., a function of \emph{only} of the state-specific count associated to each rate, respectively. The choice in the latter case comes from interpreting the counting memory as a model for capsular immunity --- this will also turn out to be quite important for our approximation further on.

Note that in Ref.~\cite{cobey2012niche}, the models of nonspecific and specific immunity suggest choosing the following functions
$$
\begin{align}
\tilde{\mu}_i( \dots, n_{i}, \dots ) &= \mu_{\rm max} + (\mu_i - \mu_{\rm max})\exp \bigg( -\epsilon \sum_{\forall i'}n_{i'} \bigg) \\
\tilde{\Lambda}_i(n_i) &= \Lambda_{i}{\sf 1}_{n_i=0} + \sigma \Lambda_{i}{\sf 1}_{n_i>0} \,.
\end{align}
$$
In the expressions above: $\epsilon$ governs the level of (immune system maturation) with respect to the number of past state occupations; ${\sf 1}_A$ denotes an indicator function whose argument is unity when $A$ is satisfied, else $0$; and the susceptibility of an individual is assumed to be reduced by a constant factor of $\sigma$ after having occupied that state once or more.

The multi-state process that we're going to consider would be normally be described as a non-Markovian phenomenon. However, the modelling approach we will take is instead a bit more similar to the formal concept of a Markov embedding (as studied in, e.g., Refs.~\cite{kanazawa2021zipf,zwanzig2001nonequilibrium}). By creating a binary state occupation variable $x_i$ for the $i$-th serotype, and the probability of occupying state $(\dots , x_i , \dots , n_i , \dots )$ at time $t$ as $P(\dots , x_i , \dots , n_i , \dots , t)$, we may write a Markovian master equation for the process. Let's now define
$$
\begin{align}
p_i(\dots ,n_i,\dots ,t) &\equiv P(\dots, x_{i}=1, x_{i'}=0, \dots ,n_{i},\dots ,t)\quad  \forall i'\neq i \\   
q(\dots ,n_i,\dots ,t) &\equiv P(\dots, x_{i}=0, \dots ,n_{i},\dots ,t) \quad \forall i\,.
\end{align}
$$
Using these definitions, it is straightforward to show that the master equation satisfies
$$
\begin{align}
\begin{aligned}
\frac{{\rm d}}{{\rm d}t}p_i(\dots ,n_i,\dots ,t) &= \tilde{\Lambda}_i(n_i-1)q(\dots ,n_{i}-1,\dots ,t) + \sum_{\forall i' \neq i}f_{i'i}\tilde{\Lambda}_i (n_i-1)p_{i'}(\dots ,n_{i'}, n_i-1,\dots ,t) \\
&\quad - \tilde{\mu}_i( \dots, n_{i}, \dots ) p_i(\dots ,n_i,\dots ,t) - \sum_{\forall i'\neq i}f_{ii'}\tilde{\Lambda}_{i'} (n_{i'}) p_i(\dots ,n_i,\dots ,t) \\
\frac{{\rm d}}{{\rm d}t}q(\dots ,n_i,\dots ,t) &= \sum_{\forall i}\tilde{\mu}_i( \dots, n_{i}, \dots )p_i(\dots ,n_i,\dots ,t) - \sum_{\forall i}\tilde{\Lambda}_i(n_i) q(\dots ,n_i,\dots ,t) \,.
\end{aligned}
\end{align}
$$
By defining the state space to encode the memory of past state occupations using the $n_i$ values themselves, the process is Markovian over the full $(\dots , x_i,\dots ,n_i,\dots)$ space. Note also that one may obtain the time-dependent joint distribution over the $(\dots ,n_i,\dots)$ space, i.e., $P(\dots, n_i, \dots, t)$, through marginalisation at any time
%%
\begin{equation}
P(\dots, n_i, \dots, t) = q(\dots, n_i, \dots, t) + \sum_{\forall i} p_i(\dots, n_i, \dots, t) \label{eq:marg} \,.   
\end{equation}
%%

Though we intend our analysis of this class of multi-state models to apply more generally beyond immediate applications to pneumococcus, it also is worth noting that restricting individuals to occupy a single state at a time only approximates the full pneumococcal carriage dynamics. The true process actually allows for some individuals to carry more than one serotype at at time. However, due to the relatively low and variable reported prevalence of simultaneous serotype carriers (or `co-colonised' individuals) across different studies~\cite{gratten1989multiple,huebner2000lack,o2003report,hare2008random,rivera2009multiplex,brugger2010multiple}, the single-state occupation model should still a good tracer model of the underlying dynamical behaviour of the system. Note also that this additional complexity in the dynamics should be straightforward to incorporate into our framework for future analyses. 
%Note also that Eq.~(\ref{eq:mast2}) has the following implicit solution
%%
%\begin{align}
%q(\dots ,n_i,\dots ,t) &= q(\dots ,n_i,\dots ,t_0)\,e^{-\sum_{\forall i}\tilde{\Lambda}_i(n_i) (t-t_0)} \nonumber \\
%& +\sum_{\forall i}\tilde{\mu}_i\big( {\textstyle \sum}_{\forall i'}n_{i'}\big)\int^{t}_{t_0}{\rm d}t'p_i(\dots ,n_i,\dots ,t') \nonumber \\ 
%& \qquad \qquad \qquad \,\, \times e^{-\sum_{\forall i}\tilde{\Lambda}_i(n_i) (t-t')} \,.
%\end{align}
%%

%As it is currently written, the permitted state space for each of the occupation histories is $n_i\in \mathbbm{Z}^{0+}$, and hence is not strictly finite. We note, however, that it is possible to approximate the dynamics with a finite ODE system for a limited period of time. More precisely, over a short enough interval in time $t-t_0$, such that the number of new states that have a chance to be visited can be restricted to a finite value, to good approximation of the exact system. If the maximum number of changes in state over the interval $t-t_0$ is truncated to a value of $N_{\rm c}$, the total number of states that can subsequently be occupied will scale according to $(N_{\rm s}+1)^{N_{\rm c}}$, where $N_{\rm s}$ is the number of serotypes. Note that, in this scaling, we have assumed a pure ensemble where there is an initially specified set of $n_i$ values at time $t_0$.

%By defining a new time-dependent vector ${\sf p}_u(t)$ of probabilities --- labelling each unique state according to the index $u$ for each available independent component of the system of Eqs.~(\ref{eq:mast1}) and~(\ref{eq:mast2}) after the truncation --- we may write the system as
%%
%\begin{equation}
%\frac{{\rm d}}{{\rm d}t}{\sf p}_u(t) = \sum_{\forall u'}{\sf M}_{uu'}{\sf p}_{u'}(t) \label{eq:ode}\,,
%\end{equation}
%%
%where the elements of ${\sf M}_{uu'}$ depend on how the state space probabilities are organised within the vector ${\sf p}_{u}(t)$. This equation has a well-known general matrix exponential solution
%%
%\begin{equation}
%\ {\sf p}_{u}(t) = \sum_{\forall u'} \bigg[ e^{{\sf M}(t-t_0)}\bigg]_{uu'}{\sf p}_{u'}(t_0) \label{eq:ode-sol}\,.
%\end{equation}
%%
%When using a pure ensemble initially such that all realisations begin in the $k$-th state, i.e., ${\sf p}_{k}(t_0)=1$, we can use Eq.~(\ref{eq:ode-sol}) to write a new equation which evolves the conditional probabilities given this initial condition, ${\sf p}_{uk}(t)$, taking the form 
%%
%\begin{equation}
%\ \frac{{\rm d}}{{\rm d}t}{\sf p}_{uk}(t) = \sum_{\forall u'}{\sf M}_{uu'}{\sf p}_{u'k}(t) = \sum_{\forall u'}{\sf p}_{uu'}(t)\,{\sf M}^{\rm T}_{u'k} \,.
%\end{equation}
%%
%If we now sum over some arbitrarily defined states of interest ${\sf P}^\Omega_{k}=\sum_{\forall u \in \Omega}{\sf p}_{uk}(t)$, and set all the remaining probabilities to 0, we can solve the adjoint form of the above equation (the second equality) to find the state retention probability (the probability of an individual starting in the $k$-th state remaining within the $\Omega$-defined set of states for the period $t-t_0$)
%%
%\begin{align}
%\ {\sf P}^\Omega_{k} &= \sum_{\forall u \in \Omega} \bigg[ e^{{\sf M}^{\rm T}(t-t_0)}\bigg]_{uk}\,.
%\end{align}
%%

%We immediately remark that by marginalising the calculated state retention probabilities above over the $n_i$ values, one may directly obtain the serotype occupation probabilities over the specified time interval $t-t_0$. By including a specified vaccination programme in the calculation, this method of evaluating the resulting serotype occupation probabilities (which does not require running many numerical realisations of a stochastic simulation, but rather the solution to a matrix exponential) can have obvious potential uses in assessing the efficacy of vaccines which cover a limited range of serotypes. This will become particularly clear when the posterior uncertainty --- over the parameters which exist in the matrix elements ${\sf M}_{uu'}^{\rm T}$ --- with respect to a given data set is included in the calculated retention probability.

Let's try an approximation for the joint distributions of $p_i(\dots, n_i, \dots, t)$ and $q(\dots, n_i, \dots, t)$ which assumes separability, such that
%%
\begin{align}
\ p_i(\dots, n_i, \dots, t) &\simeq p_i(t)P(\dots, n_i, \dots, t) \label{eq:approx1} \\
\ q(\dots, n_i, \dots, t) &\simeq q(t)P(\dots, n_i, \dots, t) \label{eq:approx2} \,.
\end{align}
%%
We shall evaluate the quality of this approximation later on (so don't worry) under different parametric conditions, but for now, let's just treat it as an ansatz.

By marginalising over states using Eqs.~(\ref{eq:mast1}) and~(\ref{eq:mast2}) in same way as shown in Eq.~(\ref{eq:marg}), then substituting in the approximations of Eqs.~(\ref{eq:approx1}) and~(\ref{eq:approx2}), and finally marginalising (each a summation from $n_{i'}=0$ to $\infty$) over the resulting relation $\forall n_{i'} \,\, i'\neq i$, one finds that the following time evolution equation is separately satisfied by each marginal $P(n_i,t)$ distribution 
%%
\begin{align}
\frac{{\rm d}}{{\rm d}t}P(n_i,t) &= \bigg[ \tilde{\Lambda}_i(n_i-1)q(t) + \sum_{\forall i'\neq i}  f_{i'i}\tilde{\Lambda}_{i} (n_{i}-1) p_{i'}(t) \bigg] P(n_{i}-1,t) \nonumber \\
&\quad - \bigg[ \tilde{\Lambda}_i(n_i)q(t) + \sum_{\forall i'\neq i} f_{ii'}\tilde{\Lambda}_{i'} (n_{i'}) p_i(t)\bigg] P(n_i,t)  \label{eq:marginal-mast} \,.
\end{align}
%%
In addition to the separability assumption, the key point which allowed us to derive this one-step marginal master equation was the dependence of $\tilde{\Lambda}_i$ on \emph{only} $n_i$; in contrast to all of the past recorded states $(\dots, n_i, \dots)$ like $\tilde{\mu}_i$.

From this point on we'll focus on the specific pneumococcus model by inserting the function definitions of Eqs.~(\ref{eq:nonspecific-imm}) and~(\ref{eq:specific-imm}) into Eq.~(\ref{eq:marginal-mast}). The \href{https://github.com/umbralcalc/pneumoinfer}{\texttt{pneumoinfer}} class is currently written for only these models (i.e., with just these choices of function), but it's useful to see how the steps above could be performed for more general models too. The solution to Eq.~(\ref{eq:marginal-mast}) with these function substitutions is simply a Poisson distribution $P(n_i,t) = {\rm Poisson}[n_i;{\rm E}_t(n_i)]$, where
%%
\begin{equation}
{\rm E}_t (n_i) = {\rm E}_{t_0}(n_i) + \int^t_{t_0}{\rm d}t'\bigg[ \sigma \Lambda_iq(t') +\sum_{\forall i'\neq i}  f_{i'i}\sigma \Lambda_{i} p_{i'}(t')\bigg] \label{eq:exN}\,.
\end{equation}
%%

Exploiting the properties of this Poisson distribution, if we now return to Eqs.~(\ref{eq:mast1}) and~(\ref{eq:mast2}) and marginalise them over all $n_i$, while noting that
%%
\begin{align}
\ p_i(t) &= \sum_{\forall n_i}\sum_{n_{i}=0}^\infty p_i(\dots, n_i, \dots, t) \\
\ q(t) &= \sum_{\forall n_i}\sum_{n_{i}=0}^\infty q(\dots, n_i, \dots, t) \,,
\end{align}
%%
one arrives at the following finite (implicitly integro-differential) system
%%
\begin{align}
\frac{{\rm d}}{{\rm d}t}p_i(t) &= \Lambda_iF_{it} q(t) + \sum_{\forall i'\neq i} f_{i'i} \Lambda_iF_{it} p_{i'}(t) - \mu_iG_{it} p_i(t)-\sum_{\forall i'\neq i}f_{ii'}\Lambda_{i'}F_{i't} p_i(t) \label{eq:igd-syst11} \\
\frac{{\rm d}}{{\rm d}t}q(t) &= \sum_{\forall i}\mu_iG_{it}p_i(t) - \sum_{\forall i}\Lambda_iF_{it}q(t)\,, \label{eq:igd-syst12}
\end{align}
%%
where, to avoid repetition, we have defined
%%
\begin{align}
\ F_{it} &= P(n_i=0,t)+\sigma P(n_i>0,t) = e^{-{\rm E}_t(n_i)}+\sigma \big[ 1-e^{-{\rm E}_t(n_i)}\big] \label{eq:igd-syst1} \\
\ G_{it} &= \frac{\mu_{\rm max}}{\mu_i} + \bigg( 1-\frac{\mu_{\rm max}}{\mu_i}\bigg) e^{\sum_{\forall i}{\rm E}_t(n_i)(e^{-\epsilon}-1)}\,, \label{eq:igd-syst2}
\end{align}
%%
where to derive $G_{it}$ we have had to assume conditional independence between $n_i$ and $n_{i'}\,\,\forall i'\neq i$. Eq.~(\ref{eq:exN}) can be differentiated to provide an equation for the time derivative of ${\rm E}_t(n_i)$ --- evolving this equation alongside Eqs.~(\ref{eq:igd-syst11}) and~(\ref{eq:igd-syst12}) yields a finite ODE system. Note also that this ODE system should apply to other forms of memory functions used in Eqs.~(\ref{eq:nonspecific-imm}) and~(\ref{eq:specific-imm}) by simply marginalising over their $n_i$ values, and so this approximate approach appears to be quite generalisable to other simlar systems.

In order to analyse the system properties and check the validity of the approach above, we're now going to make some decisions about the parameter space to explore. Let's independently draw the $(\mu_i,\Lambda_i)$ values from Gamma distributions with shapes $(\mu_\alpha,\Lambda_\alpha)$ and rates $(\mu_\beta,\Lambda_\beta)$. Let's also constrain the matrix values $f_{ii'}=f_{i}\mathbbm{I}_{i'}$ (where $\mathbbm{I}_{i'}$ denotes the elements of a simple vector of ones) which also happens to be consistent with pneumococcus data anyway~\cite{lipsitch2012estimating}. We'll also need a metric of comparison between the marginalised distribution outputs from the fully simulated master equation and our approximation. To this end, it probably makes sense to look at the Kullback-Leibler divergence~\cite{kullback1951information} between the marginal distributions for $x_i$ and $n_i$ in a full stochastic simulation and our approximation. In other words
%%
\begin{align}
\ D^{(x)}_{{}_{\rm KL}} &= \sum_{\forall i} p_{i, {\rm sim}}(t) \ln \Bigg[ \frac{p_{i, {\rm sim}}(t)}{p_i(t)} \Bigg] \nonumber \\ 
&\simeq - \sum_{\forall i} \frac{\ln Z_{\rm sim}(x_i, t)}{Z_{\rm sim}(x_i, t)} -\sum_{\forall i} \frac{\ln p_i(t)}{Z_{\rm sim}(x_i, t)} \label{eq:DKLx} \\
\ D^{(n)}_{{}_{\rm KL}} &= \sum_{\forall i} \sum_{n_i=0}^{\infty} P_{\rm sim}(\dots, n_i, \dots, t) \ln \Bigg[ \frac{P_{\rm sim}(\dots, n_i, \dots, t)}{P(n_i,t)} \Bigg] \nonumber \\ 
&\simeq - \sum_{\forall i} \sum_{n_i=0}^{\infty}\frac{\ln Z_{\rm sim}(\dots, n_i, \dots, t)}{Z_{\rm sim}(\dots, n_i, \dots, t)} -\sum_{\forall i} \sum_{n_i=0}^{\infty} \frac{\ln {\rm Poisson}[n_i;{\rm E}_t(n_i)]}{Z_{\rm sim}(\dots, n_i, \dots, t)} \label{eq:DKLn}\,,
\end{align}
%%
where $Z_{\rm sim}(\dots, x_i, \dots, t)$ and $Z_{\rm sim}(\dots, n_i, \dots, t)$ denote the marginal frequency counts in each state space derived from a stochastic simulation of the master equation. Using the \texttt{run\_sim} method of the \href{https://github.com/umbralcalc/pneumoinfer}{\texttt{pneumoinfer}} class, and Eqs.~(\ref{eq:DKLx}) and~(\ref{eq:DKLn}), we can generate numerically-approximate plots of the Kullback-Leibler divergence on the marginal distributions over a range of numbers of states, parameters and points in time.

\textcolor{red}{Need to make lots of comparison plots here with the full simulation to verify the results of this section...}

\subsection{Varying $\Lambda_{iu}$ model}

\textcolor{red}{Code this up and more testing plots needed for this section too. }

We're now ready to introduce an alternative model which accounts for a stochastically-varying susceptibility $\Lambda_{iu}$ (a possible model for community exposure to infectious individuals), which is now additionally indexed by `$u$' which corresponds to each individual. In this model, we have 
%%
\begin{equation}
\Lambda_{iu} = \Lambda_{\rm min} + \lambda\sum_{\forall u'\neq u}\beta_{uu'} \frac{x_{iu'}}{N_{\rm p}}\,,
\end{equation}
%%
where: the total population number is $N_{\rm p}$; $\beta_{uu'}$ are elements of a `contact matrix' which rescales the event rate according to the spreading behaviour between the $u$-th and $u'$-th individuals; $\lambda$ is a constant normalisation for $\beta_{uu'}$; and $x_{iu'}$ records the state of the $u'$-th individual.

Extending Eqs.~(\ref{eq:mast1}) and~(\ref{eq:mast2}) to include the susceptibility above and the states of $N_{\rm p}$ individuals, one can easily adapt the argument of the previous section to arrive at the following generalisation of the system described by Eqs.~(\ref{eq:igd-syst11}) and~(\ref{eq:igd-syst12})
%%
\begin{align}
\frac{{\rm d}}{{\rm d}t}p_{iu}(t) &= {\rm E}_t(\Lambda_{iu})F_{it} q_u(t) + \sum_{\forall i'\neq i} f_{i'i} {\rm E}_t(\Lambda_{iu})F_{it} p_{i'u}(t) \nonumber \\
&\quad - \mu_iG_{it} p_{iu}(t)-\sum_{\forall i'\neq i}f_{ii'}{\rm E}_t(\Lambda_{i'u})F_{i't} p_{iu}(t) \label{eq:igd-syst11-gen} \\
\frac{{\rm d}}{{\rm d}t}q_u(t) &= \sum_{\forall i}\mu_iG_{it}p_{iu}(t) - \sum_{\forall i}{\rm E}_t(\Lambda_{iu})F_{it}q_u(t)\,. \label{eq:igd-syst12-gen}
\end{align}
%%
In the equations above, the state occupation probabilities of separate $u$-indexed individuals (or $u$-indexed categories of individual) are $p_{iu}(t)$ and $q_u(t)$, and we've also computed the expectation
%%
\begin{equation}
{\rm E}_t(\Lambda_{iu}) = \Lambda_{\rm min} + \lambda\sum_{\forall u'\neq u}\beta_{uu'} \frac{p_{iu'}(t)}{N_{\rm p}}\,.
\end{equation}
%%
