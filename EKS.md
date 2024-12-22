---


---

<h1 id="build-your-own-eks-cluster-with-autoscaling-hpa--karpenter">Build Your Own EKS Cluster with Autoscaling (HPA + Karpenter)</h1>
<blockquote>
<p><strong>“The grand finale is a wallet-saving move—skip it, and your wallet might file for bankruptcy!”</strong><br>
<em>Don’t get it? Let me rephrase: If you care for your money, remember to execute the last step in the blog (Deleting the cluster!). DON’T blame me for your carelessness.</em></p>
</blockquote>
<hr>
<h2 id="welcome-to-the-guide">Welcome to the Guide!</h2>
<p><em>“Welcome, dear reader, to the ultimate guide on deploying an EKS cluster with the magic wand of Kubernetes wizards—eksctl! If you’ve ever looked at cloud computing and thought, ‘Wow, this feels like assembling IKEA furniture without instructions,’ don’t worry—you’re not alone. But by the end of this blog, you’ll be deploying like a pro, and who knows, you might even find it fun (or at least less painful than putting together that bookshelf).”</em></p>
<hr>
<h2 id="step-1-installing-aws-cli">Step 1: Installing AWS CLI</h2>
<p><em>“First things first, download and install the AWS CLI. Think of it as your universal remote for the AWS world. Without it, you’re like a wizard without a wand—tragic and completely useless. Head to the <a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html">AWS CLI installation guide</a> and follow the instructions.”</em></p>
<p><strong>WAIT!</strong><br>
You don’t want to browse away and risk missing out, right? Let me make it easy for you, my dear Linux users 😊.</p>
<h3 id="install-aws-cli-linux-users">Install AWS CLI (Linux Users)</h3>
<p>Here’s the exact set of commands you’ll need to install AWS CLI on your Linux machine:</p>
<pre class=" language-bash"><code class="prism  language-bash">curl <span class="token string">"https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"</span> -o <span class="token string">"awscliv2.zip"</span>
unzip awscliv2.zip
<span class="token function">sudo</span> ./aws/install
</code></pre>

