---
title: "Google Online Challenges"
slug: google-online-challenges
tags: [algorithms, interview questions]
published: 2019-10-07
---

Last Friday (4 Oct 2019), I completed Google's online challenge questions. There were 2 algorithmic questions and the time limit to solve both questions is 45 mins.

I managed to solve first question in 30 mins but wasn't able to solve the second question until after the time is up. The following are the 2 questions in summary as I recall from memory. I included my solutions to both questions but do note that the while the solutions pass the basic test cases, I cannot guarantee if they pass any hidden complex test cases.

## Question 1 - Berry Picking
Program a robot to pick the max no of berries from bushes. Each bush contains `berries[i]` berries. Robot has a limit of `limit` number of berries he can carry. Robot must take all berries from a bush. Robot cannot take berries from any adjacent bushes.

Output max no of berries the robot can take.

Example:
- berries per bush (`berries[]`) = [50, 10, 20, 30, 40]
- robot's carry limit (`limit`) = 100
- answer: 90
- explanation: choose 50 and 40

I googled and found a [similar problem that is solved using dynamic programming](https://stackoverflow.com/questions/4487438/maximum-sum-of-non-consecutive-elements) but the solution wouldn't work for this question cos this question has limit and can jump around in the array.

### Pseudocode c++ solution:
{% highlight c++ linedivs %}
  vector<pair<int,int>> map
  vector<int> adj
  // loop through berries[]:
    map.first = i
    map.second = berries[i]
  // sort map in descending order of second value
  sum = 0
  // loop through map
    if sum + map.second > limit
      continue
    if adj doesnt contain pair.first
      adj add pair.first - 1 and pair.first + 1
      sum += pair.second
  print sum
{% endhighlight %}

I'm skipping the explanations for this question because I think it's quite straightforward.

## Question 2 - Optimise Salaries
Optimise a company's employment costs. Each month the company need `employees[i]` employees. Start with 0 employee. Given employee hiring cost, employee's per month salary, employee's severence (firing) cost, output minimum sum of costs.

Example:
- hire cost (`h`) = 4
- salary cost (`s`) = 5
- fire cost (`f`) = 6
- employees needed per month (`employees[]`) = [11, 9, 10, 14, 9]
- answer: 361
- explanation:
  - actual employees count per month = [11, 10, 10, 14, 14]
  - first month hire 11 people then fire 1
  - second month no change
  - third month no change
  - fourth month hire 3 people
  - fifth month no change

I struggled with this question. The time when I first read the question was 11 40am. I solved it locally after the end of the test at 1 26pm. Took me almost 2 hours. The problem I struggled with was "how to decide when to fire a person". Hiring is easy - when there is not enough people, just hire. But firing cannot be decided by only looking at the current month's need.

### Solution explanation
It's quite difficult for me to explain my train of thought, but I will try. Let's analyse the following scenario.

`employees[] = [10, 9, 10]`

By end of first month, there are 10 employees. Now we need to decide whether to fire 1 person or keep at 10.

Scenario 1: fire then hire
- cost of scenario 1 `c1` involves:
  - cost for second month = `f * 1 + s * 9`
  - cost for third month = `h * 1 + s * 10`

Scenario 2: keep at 10 person
- cost of scenario 2 `c2` involves:
  - cost for second month = `s * 10`
  - cost for third month = `s * 10`

Notice the difference: `c1 = c2 + 1 * f + 1 * h - 1 * s`

So the extra cost for firing then hiring is a 1 time fire and hire fee for `p` persons fired, offset with the salary saved for `p` persons fired times `m` months.

Let `d` be the cost difference for scenario 1 and 2: `c1 - c2 = p * (f + h) - p * m * s`
- if `d > 0`, `c1 > c2`, `p*(f+h) > p*m*s`, use c2
- if `d < 0`, `c1 < c2`, `p*(f+h) < p*m*s`, use c1
- if `d == 0`, `c1 = c2`, `p*(f+h) == p*m*s`, use either

This means we need to compare the fire and hire fee with the salary saved, and the variables to consider are the number of people fired and months of salary saved. In fact, we can eliminate `p` (no of people fired) from the equation, so we just need to compare `f+h` and `m*s`.

To test the significance of `p`, compare the first case with `employees[] = [10, 8, 10]`
- `c1 = c2 + 2 * f + 2 * h - 2 * s`
- To decide, we still also need to compare `2*(f+h)` and `2*s` which is basically comparing fire+hire and salary
  but things are different if the months of salary saved is different

Instead we can test for `m`, compare the first case with `employees[] = [10, 9, 9, 10]`
- `c1 = c2 + 1 * f + 1 * h - 2 * s`
- To decide, we need to compare `1*(f+h) and 2*s`

This means that the decision depends on the number of months of salary saved. So we need to lookahead and count a certain number of months. We can calculate the number of months to lookahead based on the equation.

`f+h < m*s`, use c1

`m > (f+h)/s`

So for the next `m` months, if you can fire people, fire just enough people.

Example:
- `employees[] = [10, 9, 9, 10]`, `f = 7`, `h = 4`, `s = 5`
- So `(f+h)/s = 2`, if `m > 2`, then fire
- This means that we need to look ahead for the next 3 months (because `m > 2`)

The following shows comparison between cost of scenario 1 `c1` and cost of scenario 2 `c2`.
{% highlight plaintext linedivs %}
em:   10  9  9 10
c1:   90 52 45 54 = 241
c2:   90 50 50 50 = 240
{% endhighlight %}

So at the end of the every month, we look at the next 3 months and get the max no of employees needed, then fire minimally.

### Code solution
Original buggy solution in c++:
{% highlight c++ linedivs %}
#include <bits/stdc++.h>
using namespace std;

int main() {
  int hire, salary, sever, months;
  cin >> hire >> salary >> sever >> months;
  int count[months];
  for (int i = 0; i < months; i++) {
    cin >> count[i];
  }
  // extra month salary if sever then hire
  int a = (sever - salary + hire) / salary;
  int employees = 0;
  int sum = 0;
  for (int i = 0; i < months; i++) {
    // fire people
    int fire = 0;
    int max = 0;
    int j;
    for (j = i + 1; j < months && j <= i + a + 1; j++) {
      if (count[j] > max) {
        max = count[j];
      }
    }
    if (max < employees && i < months - a - 1) {
      sum += sever * (employees - max); // sever this no of people
    }
    cout << " employees: " << employees << '\n';
    if (employees < count[i]) { // hire people
      sum += hire * (count[i] - employees);
      employees = count[i];
    }
    sum += salary * employees; // salary
  }
  cout << sum << '\n';
  return 0;
}
{% endhighlight %}

As I was writing this post, formulating the logic above, I noticed some mistakes in my code. Specifically, I did not update the `employees` count in the line 25 if block. I also could have avoided the off-by-1s if I remove `- salary` on line 12. Also, I rearranged the sequence in the loop such that it sums hiring cost, then current month's salary, then firing cost.

Revised solution in c++:
{% highlight c++ linedivs %}
#include <bits/stdc++.h>
using namespace std;

int main() {
  int hire, salary, fire, months;
  cin >> hire >> salary >> fire >> months;
  int employees[months];
  for (int i = 0; i < months; i++) {
    cin >> employees[i];
  }
  int m = (fire + hire) / salary; // no of months to lookahead
  cout << "m: " << m << '\n';
  int curr_employee = 0;
  int sum = 0;
  for (int i = 0; i < months; i++) {
    cout << " curr_employee: " << curr_employee << '\n';
    // hire people
    if (curr_employee < employees[i]) {
      cout << "hiring " << (employees[i] - curr_employee) << '\n';
      sum += hire * (employees[i] - curr_employee);
      curr_employee = employees[i];
    }
    // monthly salary
    sum += salary * curr_employee;
    // fire people
    int max = 0;
    int j;
    for (j = i + 1; j < months && j <= i + m + 1; j++) {
      if (employees[j] > max) {
        max = employees[j];
      }
    }
    if (max < curr_employee && i < months - m) {
      int fire = curr_employee - max; // fire this no of people
      cout << "firing " << fire << '\n';
      sum += fire * fire;
      curr_employee = max;
    }
  }
  cout << sum << '\n'; // answer
  return 0;
}
{% endhighlight %}

Test case 1 (using original example):
{% highlight plaintext linedivs %}
4 5 6 5
11 9 10 14 9
{% endhighlight %}

Output:
{% highlight plaintext linedivs %}
m: 2
 curr_employee: 0
hiring 11
 curr_employee: 11
 curr_employee: 11
 curr_employee: 11
hiring 3
 curr_employee: 14
361
{% endhighlight %}

361 is the correct answer.