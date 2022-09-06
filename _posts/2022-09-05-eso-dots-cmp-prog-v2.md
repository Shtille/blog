---
layout: post
title: "ESO DoTs comparison program. Part 2"
author: "Shtille"
categories: journal
tags: [C++,samples,ESO]
---

Check the [previous article]({{ 'eso-dots-comparison' | relative_url }}) to recall.

## Test program for dots comparison

### Program code

Skill names you can check at [ESO skillbook](https://eso-skillbook.com/skilltree/templar).

```cpp
#include <string>
#include <iostream>
#include <memory>
#include <list>

class Skill {
public:
	Skill(const std::string& name, double base_damage, double duration)
	: name_(name)
	, base_damage_(base_damage)
	, duration_(duration)
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
		return GetDamage() / GetDuration();
	}
	const std::string& GetName() const
	{
		return name_;
	}
protected:
	std::string name_;
	double base_damage_;
	double duration_;
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
	: LastingSkill("Carve", 7097., 0., 1., 12708., 12.)
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
	bool result = (R3-R2)+((T3-T2)*(R1*T1)/(T2*T3)) > 0.;
	if (print)
	{
		const Skill *first, *second;
		if (result)
			second = skill2, first = skill3;
		else
			first = skill2, second = skill3;
		std::cout << "skill " << first->GetName() << 
			" is better than " << second->GetName() << std::endl;
	}
	return result;
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
		std::make_shared<Carve>()
	};
	// Declare comparator
	ListComparator comparator(spammable.get());
	list.sort(comparator);
	// Print results
	std::cout << spammable->GetName();
	for (auto& skill_ptr : list)
	{
		std::cout << " > " << skill_ptr->GetName();
	}
	std::cout << std::endl;
	return 0;
}
```

### Program output

Single line output has been divided by several lines.

```
Biting Jabs > Stampede > Ritual of Retribution > Deadly Cloak > Carve > 
Vampire's Bane > Degeneration > Blazing Spear > Barbed Trap > 
Consuming Trap > Solar Barrage > Rending Slashes
```

### Conclusion

It's funny that skill with lower DPS _Vampire's Bane_ has more outcome than _Blazing Spear_ with higher DPS.

### Improvements

- Add additional skills.
- Try different base skill (spammable) and compare results.

## Skills to add

- Power of the Light 
- Radiant Oppression ?
- Lethal Arrow (bow spam)
- Endless Hail
- Acid Spray 
- Poison Injection 
- Dizzying Swing (2h spam)
- Scalding Rune
- Crushing Weapon (psijic)
- Shadow Silk
- Mystic Orb