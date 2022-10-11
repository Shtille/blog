---
layout: post
title: "ESO DoTs comparison program. Part 3"
author: "Shtille"
categories: journal
tags: [C++,samples,ESO]
---

Check the [previous article]({{ 'eso-dots-cmp-prog-v2' | relative_url }}) to recall.

## Test program for dots comparison

In this version I've added some new skills including _Poison Injection_ that damage increases as foe's health goes down.
Added comparison of 3 skills builds to compare _Maelstrom Arena Bow_ and _Maelstrom Arena Sword_.

### Program code

```cpp
#include <string>
#include <iostream>
#include <memory>
#include <list>
#include <format>

struct MaelstromArenaBow {
	static constexpr double kBaseAddition = 430.;
	static constexpr double kTickAddition = 191.;
	static constexpr int kMaxNumberAddition = 8;
};

class Skill {
public:
	Skill(const std::string& name, double base_damage, double duration)
	: name_(name)
	, base_damage_(base_damage)
	, duration_(duration)
	, stored_damage_(0.)
	, stored_(false)
	{
	}
	virtual ~Skill() = default;
	virtual double GetDamage() const
	{
		return base_damage_;
	}
	virtual double GetDuration() const
	{
		return duration_;
	}
	double GetDps() const
	{
		if (!stored_)
		{
			stored_damage_ = GetDamage();
			stored_ = true;
		}
		return stored_damage_ / GetDuration();
	}
	const std::string& GetName() const
	{
		return name_;
	}
protected:
	std::string name_;
	double base_damage_;
	double duration_;
private:
	mutable double stored_damage_;
	mutable bool stored_;
};

class InstantSkill : public Skill {
public:
	InstantSkill(const std::string& name, double base_damage)
	: Skill(name, base_damage, 1.)
	{
	}
};

class LastingSkill : public Skill {
public:
	LastingSkill(
		const std::string& name,
		double base_damage, 
		double damage_tick, 
		double time_tick,
		double damage_over_duration,
		double duration,
		double prepare_time = 0.)
	: Skill(name, base_damage, duration)
	, damage_tick_(damage_tick)
	, time_tick_(time_tick)
	, damage_over_duration_(damage_over_duration)
	, prepare_time_(prepare_time)
	{
	}
	virtual double GetDamage() const
	{
		return base_damage_ 
			+ (damage_tick_ / time_tick_) * duration_ 
			+ damage_over_duration_;
	}
	virtual double GetDuration() const
	{
		return duration_ + prepare_time_;
	}
protected:
	double damage_tick_;
	double time_tick_;
	double damage_over_duration_;
	double prepare_time_;
};

class FinishingLastingSkill : public LastingSkill {
public:
	FinishingLastingSkill(
		const std::string& name,
		double base_damage, 
		double damage_tick, 
		double time_tick,
		double damage_over_duration,
		double duration,
		double prepare_time,
		double health_threshold,
		double finishing_multiplier)
	: LastingSkill(
		name,
		base_damage,
		damage_tick,
		time_tick,
		damage_over_duration,
		duration,
		prepare_time)
	, health_threshold_(health_threshold)
	, finishing_multiplier_(finishing_multiplier)
	{
	}
	virtual double GetDamage() const
	{
		double base_dps = (damage_tick_ / time_tick_) + 
			(base_damage_ + damage_over_duration_) / duration_;
		double dps = base_dps * (0.5 * health_threshold_ * finishing_multiplier_ + 1);
		return dps * duration_;
	}
private:
	double health_threshold_;
	double finishing_multiplier_;
};

class LastingSkillWithTickAddition : public LastingSkill {
public:
	LastingSkillWithTickAddition(
		const std::string& name,
		double base_damage, 
		double damage_tick, 
		double time_tick,
		double damage_over_duration,
		double duration,
		double prepare_time,
		double base_addition,
		double tick_addition,
		int max_count_addition)
	: LastingSkill(
		name,
		base_damage,
		damage_tick,
		time_tick,
		damage_over_duration,
		duration,
		prepare_time)
	, base_addition_(base_addition)
	, tick_addition_(tick_addition)
	, max_count_addition_(max_count_addition)
	{
	}
	virtual double GetDamage() const
	{
		double nd = duration_ / time_tick_;
		int n = static_cast<int>(nd);
		int n_inc = (n < max_count_addition_) ? n : max_count_addition_;
		int n_max = (n > max_count_addition_) ? (n - max_count_addition_) : 0;
		double damage_increase = 0.5 * (2. * base_addition_ + tick_addition_ * (n_inc - 1)) * n_inc;
		double damage_max = (base_addition_ + tick_addition_ * (max_count_addition_ - 1)) * n_max;
		return base_damage_ 
			+ ((damage_tick_ / time_tick_) * duration_) 
			+ damage_over_duration_ + damage_increase + damage_max;
	}
private:
	double base_addition_;
	double tick_addition_;
	int max_count_addition_;
};

class DelayedSkill : public Skill {
public:
	DelayedSkill(const std::string& name, double base_damage, 
		double delayed_damage, double duration)
	: Skill(name, base_damage, duration)
	, delayed_damage_(delayed_damage)
	{
	}
	virtual double GetDamage() const
	{
		return base_damage_ + delayed_damage_;
	}
protected:
	double delayed_damage_;
};

// ---- Skills definition ----

class BitingJabs final : public InstantSkill {
public:
	BitingJabs()
	: InstantSkill("Biting Jabs", 4276.*3.)
	{}
};

class DeadlyCloak final : public LastingSkill {
public:
	DeadlyCloak()
	: LastingSkill("Deadly Cloak", 0., 2505., 2., 0., 20.)
	{}
};

class BlazingSpear final : public LastingSkill {
public:
	BlazingSpear()
	: LastingSkill("Blazing Spear", 9530., 803., 1., 0., 10.)
	{}
};

class BarbedTrap final : public LastingSkill {
public:
	BarbedTrap()
	: LastingSkill("Barbed Trap", 5348., 0., 1., 17280., 20., 1.5)
	{}
};

class SolarBarrage final : public LastingSkill {
public:
	SolarBarrage()
	: LastingSkill("Solar Barrage", 0., 1865., 2., 0., 22.)
	{}
};

class VampiresBane final : public LastingSkill {
public:
	VampiresBane()
	: LastingSkill("Vampire's Bane", 4731., 0., 1., 25424., 32.)
	{}
};

class RapidStrikes final : public InstantSkill {
public:
	RapidStrikes()
	: InstantSkill("Rapid Strikes", 3207.*4.3)
	{}
};

class RendingSlashes final : public LastingSkill {
public:
	RendingSlashes()
	: LastingSkill("Rending Slashes", 2838., 0., 1., 15390., 20.)
	{}
};

class RitualOfRetribution final : public LastingSkill {
public:
	RitualOfRetribution()
	: LastingSkill("Ritual of Retribution", 0., 1926., 2., 0., 20.)
	, damage_increase_(0.12)
	{}
	virtual double GetDamage() const
	{
		double dps_increase = 0.5 * (damage_tick_ / time_tick_) 
			* damage_increase_ * (duration_ / time_tick_ - 1.);
		return base_damage_ 
			+ ((damage_tick_ / time_tick_ + dps_increase) * duration_) 
			+ damage_over_duration_;
	}
private:
	double damage_increase_;
};

class Degeneration final : public LastingSkill {
public:
	Degeneration()
	: LastingSkill("Degeneration", 0., 0., 1., 24618., 24.)
	{}
};

class ConsumingTrap final : public LastingSkill {
public:
	ConsumingTrap()
	: LastingSkill("Consuming Trap", 0., 0., 1., 20515., 20.)
	{}
};

class Stampede final : public LastingSkill {
public:
	Stampede()
	: LastingSkill("Stampede", 5677., 1413., 1., 0., 15.)
	{}
};

class Carve final : public LastingSkill {
public:
	Carve()
	: LastingSkill("Carve", 7097., 0., 1., 12708.*(32./12.), 32.)
	{}
};

class PoisonInjection final : public FinishingLastingSkill {
public:
	PoisonInjection()
	: FinishingLastingSkill("Poison Injection", 4731., 0., 1., 15390., 20., 0., 0.5, 1.2)
	{}
};

class EndlessHail final : public LastingSkillWithTickAddition {
public:
	EndlessHail()
	: LastingSkillWithTickAddition("Endless Hail", 0., 1520., 1., 0., 13., 2., 
		0., 0., 0)
	{}
};

class EndlessHailWithBow final : public LastingSkillWithTickAddition {
public:
	EndlessHailWithBow()
	: LastingSkillWithTickAddition("Endless Hail with Bow", 0., 1520., 1., 0., 13., 2., 
		MaelstromArenaBow::kBaseAddition, MaelstromArenaBow::kTickAddition, 
		MaelstromArenaBow::kMaxNumberAddition)
	{}
};

class ArrowBarrage final : public LastingSkillWithTickAddition {
public:
	ArrowBarrage()
	: LastingSkillWithTickAddition("Arrow Barrage", 0., 1976., 1., 0., 8., 2., 
		0., 0., 0)
	{}
};

class ArrowBarrageWithBow final : public LastingSkillWithTickAddition {
public:
	ArrowBarrageWithBow()
	: LastingSkillWithTickAddition("Arrow Barrage with Bow", 0., 1976., 1., 0., 8., 2., 
		MaelstromArenaBow::kBaseAddition, MaelstromArenaBow::kTickAddition, 
		MaelstromArenaBow::kMaxNumberAddition)
	{}
};

class StampedeWithModifier final : public LastingSkill {
public:
	StampedeWithModifier()
	: LastingSkill("Stampede with Sword", 5677., 1413., 1., 0., 15.)
	, mod_damage_(7051.)
	{}
	virtual double GetDamage() const
	{
		return base_damage_ 
			+ (damage_tick_ / time_tick_) * duration_ 
			+ damage_over_duration_ + mod_damage_;
	}
private:
	double mod_damage_;
};

class PowerOfTheLight final : public DelayedSkill {
public:
	PowerOfTheLight()
	: DelayedSkill("Power of the Light", 4731., 13920., 6.)
	{}
};

class ShadowSilk final : public DelayedSkill {
public:
	ShadowSilk()
	: DelayedSkill("Shadow Silk", 7174., 9566., 10.)
	{}
};

class ScaldingRune final : public LastingSkill {
public:
	ScaldingRune()
	: LastingSkill("Scalding Rune", 9463., 0., 1., 12309., 22.)
	{}
};

class Caltrops final : public LastingSkill {
public:
	Caltrops()
	: LastingSkill("Caltrops", 0., 1217., 1., 0., 10.)
	{}
};

class FlawlessDawnbreaker final : public LastingSkill {
public:
	FlawlessDawnbreaker()
	: LastingSkill("Flawless Dawnbreaker", 12944., 0., 1., 16737., 6.)
	{}
};

// ---- End of skills definition ----

/**
 * Function compares skill sets (1,2) and (1,3).
 * @param[in] skill1 		The base skill (spammable).
 * @param[in] skill2 		The skill to test from skillset (1,2).
 * @param[in] skill3 		The skill to test from skillset (1,3).
 * @param[in] print 		Whether need to print the result.
 * @return Returns true if skillset (1,3) is better than (1,2), and false otherwise.
 */
bool CompareSkillSets(const Skill* skill1, const Skill* skill2, 
	const Skill* skill3, bool print)
{
	// Define variables
	double T1 = skill1->GetDuration();
	double T2 = skill2->GetDuration();
	double T3 = skill3->GetDuration();
	double R1 = skill1->GetDps();
	double R2 = skill2->GetDps();
	double R3 = skill3->GetDps();
	return (R3-R2)+((T3-T2)*(R1*T1)/(T2*T3)) > 0.;
}
void CompareThreeSkillSets(const Skill* skill1, 
	const Skill* skill2, const Skill* skill3, int n2, int n3,
	const Skill* skill4, const Skill* skill5, int n4, int n5,
	bool use_correction)
{
	// Define variables
	double T1 = skill1->GetDuration();
	double T2 = skill2->GetDuration();
	double T3 = skill3->GetDuration();
	double T4 = skill4->GetDuration();
	double T5 = skill5->GetDuration();
	double T = static_cast<double>(n3) * T3;
	double R1 = skill1->GetDps();
	double R2 = skill2->GetDps();
	double R3 = skill3->GetDps();
	double R4 = skill4->GetDps();
	double R5 = skill5->GetDps();
	double d1_rem = 0., d2_rem = 0.;
	if (use_correction)
	{
		int t2_rem = static_cast<int>(T-T1) % static_cast<int>(T2);
		int t4_rem = static_cast<int>(T-T1) % static_cast<int>(T4);
		d1_rem = (t2_rem == 0) ? 0. : R2*(T2-static_cast<double>(t2_rem));
		d2_rem = (t4_rem == 0) ? 0. : R4*(T4-static_cast<double>(t4_rem));
	}
	double D1 = R1*(T-((T-T1)/T2)*T1-(T/T3)*T1)+R2*(T-T1)+R3*T-d1_rem;
	double D2 = R1*(T-((T-T1)/T4)*T1-(T/T5)*T1)+R4*(T-T1)+R5*T-d2_rem;
	std::cout << "Compare weapon skills:" << std::endl;
	std::cout << "First:  (" << D1 << ") " << 
		skill2->GetName() << " + " << skill3->GetName() << std::endl;
	std::cout << "Second: (" << D2 << ") " << 
		skill4->GetName() << " + " << skill5->GetName() << std::endl;
	if (D1 == D2)
		std::cout << "Equal!" << std::endl;
	else if (D1 < D2) {
		std::string result = std::format("Second is better ({} < {})", D1, D2);
		std::cout << result << std::endl;
	} else {
		std::string result = std::format("First is better ({} > {})", D1, D2);
		std::cout << result << std::endl;
	}
}

class ListComparator {
public:
	ListComparator(const Skill * spammable)
	: spammable_(spammable)
	{}

	bool operator()(const std::shared_ptr<Skill>& s1, const std::shared_ptr<Skill>& s2)
	{
		// Make descending order
		return !CompareSkillSets(spammable_, s1.get(), s2.get(), false);
	}

private:
	const Skill * spammable_;
};

int main(int argc, char const *argv[])
{
	auto spammable = std::make_unique<BitingJabs>();
	// Declare list of skills
	std::list< std::shared_ptr<Skill> > list = {
		std::make_shared<DeadlyCloak>(),
		std::make_shared<BlazingSpear>(),
		std::make_shared<BarbedTrap>(),
		std::make_shared<SolarBarrage>(),
		std::make_shared<VampiresBane>(),
		std::make_shared<RendingSlashes>(),
		std::make_shared<RitualOfRetribution>(),
		std::make_shared<Degeneration>(),
		std::make_shared<ConsumingTrap>(),
		std::make_shared<Stampede>(),
		std::make_shared<Carve>(),
		std::make_shared<EndlessHail>(),
		std::make_shared<EndlessHailWithBow>(),
		std::make_shared<ArrowBarrage>(),
		std::make_shared<ArrowBarrageWithBow>(),
		std::make_shared<StampedeWithModifier>(),
		std::make_shared<PoisonInjection>(),
		std::make_shared<PowerOfTheLight>(),
		std::make_shared<ShadowSilk>(),
		std::make_shared<ScaldingRune>(),
		std::make_shared<Caltrops>(),
		std::make_shared<FlawlessDawnbreaker>()
	};
	// Declare comparator
	ListComparator comparator(spammable.get());
	list.sort(comparator);
	// Print results
	std::cout << spammable->GetName() << std::endl;
	for (auto& skill_ptr : list)
	{
		std::cout << " > " << skill_ptr->GetName() << std::endl;
	}
	std::cout << std::endl;
	// Compare Bow and Sword skills
	{
		EndlessHailWithBow skill3;
		PoisonInjection skill2;
		StampedeWithModifier skill5;
		Carve skill4;
		int n2 = 12;
		int n3 = 15;
		int n4 = 7;
		int n5 = 15;
		CompareThreeSkillSets(spammable.get(),
			&skill2, &skill3, n2, n3,
			&skill4, &skill5, n4, n5,
			false);
	}
	return 0;
}
```

### Program output

```
Biting Jabs
 > Flawless Dawnbreaker
 > Endless Hail with Bow
 > Stampede with Sword
 > Arrow Barrage with Bow
 > Power of the Light
 > Stampede
 > Carve
 > Ritual of Retribution
 > Poison Injection
 > Deadly Cloak
 > Vampire's Bane
 > Degeneration
 > Blazing Spear
 > Endless Hail
 > Barbed Trap
 > Scalding Rune
 > Shadow Silk
 > Consuming Trap
 > Solar Barrage
 > Arrow Barrage
 > Rending Slashes
 > Caltrops

Compare weapon skills:
First:  (3.40391e+06) Poison Injection + Endless Hail with Bow
Second: (3.39982e+06) Carve + Stampede with Sword
First is better (3403913.16 > 3399824)
```

Let's change spammable to _Rapid Strikes_ and retry.

```
Rapid Strikes
 > Flawless Dawnbreaker
 > Endless Hail with Bow
 > Stampede with Sword
 > Arrow Barrage with Bow
 > Stampede
 > Carve
 > Power of the Light
 > Ritual of Retribution
 > Poison Injection
 > Deadly Cloak
 > Vampire's Bane
 > Degeneration
 > Barbed Trap
 > Endless Hail
 > Blazing Spear
 > Scalding Rune
 > Consuming Trap
 > Solar Barrage
 > Shadow Silk
 > Rending Slashes
 > Arrow Barrage
 > Caltrops

Compare weapon skills:
First:  (3.59518e+06) Poison Injection + Endless Hail with Bow
Second: (3.59513e+06) Carve + Stampede with Sword
First is better (3595178.6399999997 > 3595130.3)
```

As you see the order has changed.

Concerning choice of maelstrom weapon: as you may see, the damage is almost the same. So it's up to you what to choose. Anyway _Stampede_ starter damage is guaranteed to be critical hit. Thus I'd choose the sword.

## Appendix

### Function to calculate the period for weapon skills comparison

```cpp
/**
 * @brief Calculates repeat times for primary and secondary skill.
 * The first skill is spammable.
 * The second skill is secondary skill.
 * The third skill is primary skill.
 * 
 * @param[in] t1 The first skill duration.
 * @param[in] t2 The second skill duration.
 * @param[in] t3 The third skill duration.
 * @param[out] n2 Repeat times for second skill.
 * @param[out] n3 Repeat times for third skill.
 * @return True if successfully found the solution and false otherwise.
 */
bool find_solution(int t1, int t2, int t3, int *n2, int *n3)
{
    int x,y;
    int pos = 0;
    int count = 0;
    while (count < 1000) {
        pos += t3;
        ++count;
        if ((pos - t1) % t2 == 0) {
            *n2 = (pos - t1) / t2;
            *n3 = pos / t3;
            return true;
        }
    }
    return false;
}
```